# ⚡ LND Migration Guide (Raspiblitz, Bare-metal / MiniBolt)
# NOT YET TESTED
---

> This document describes a **migration process**, not a drop-in configuration.
> Paths, firewall rules, and `lnd.conf` settings must be adapted to each
> operator’s environment.


## 0. Assumptions & Rules (READ FIRST)

**Assumptions**

* Source node: Raspiblitz
* Target node: MiniBolt (bare metal)
* LND uses its embedded BoltDB-based data store (including wallet.db and channel state).
* No Postgres
* Bitcoin Core already running on MiniBolt
* MiniBolt LND version >= Raspiblitz LND version

**Hard Rules**

* LND must **never run on both machines**
* Old node must be **physically prevented** from starting LND
* Migration is **forward-only** (no downgrading after start)
* SCB is exported multiple times
* Automation stays disabled until manual validation is complete

---

## 1. Version Handling (Important Context)

**What this does**
Clarifies how LND version differences are handled.

**Why this matters**
LND supports **forward database migrations**, not backward ones.

**Policy**

* We intentionally migrate into a **newer LND**
* LND will upgrade its embedded DB on first start
* Once started, **do not downgrade LND**

✅ This is supported behavior
✅ This is what Raspiblitz upgrades do internally
✅ This avoids a double-migration later

---

## 2. Phase 0 — Pre-Flight Checks (No Risk, Offline)

### What this does

Confirms data, config, and environment are sane *before* touching anything.

### Why it matters

Catches wiring errors early and safely.

### Commands

**On Raspiblitz**

```bash
sudo tree -d -L 5 /mnt/hdd/lnd
sudo ls -lh /mnt/hdd/lnd/data/chain/bitcoin/mainnet
```

You should see:

* `wallet.db`
* `channel.backup`
* macaroons

**On MiniBolt**

```bash
sudo find /data/lnd -maxdepth 5 -type d
sudo ls -lh /data/lnd/data/chain/bitcoin/mainnet
```

### Success looks like

* Directory structures match internally
* Files exist with expected sizes
* No references to `/mnt/hdd` on MiniBolt

---

## 3. Phase 1 — Freeze Old Node (Hard Lock)

### What this does

Permanently prevents the old node from starting LND accidentally.

### Why it matters

This removes *all* dual-node risk, even from hidden services or human error.

### Commands (Raspiblitz)

```bash
sudo systemctl stop lnd
sudo -u bitcoin lncli exportchanbackup --all
sudo mv /mnt/hdd/lnd /mnt/hdd/lnd.FROZEN-$(date +%Y%m%d)
```

### Validation

```bash
ls /mnt/hdd | grep lnd.FROZEN
systemctl status lnd
```

### Success looks like

* `/mnt/hdd/lnd` no longer exists
* LND cannot possibly start
* Data preserved intact

⚠️ **If `/mnt/hdd/lnd` still exists → STOP**

---

## 4. Phase 2 — Prepare MiniBolt (Rollback Safety)

### What this does

Creates a full rollback point on the new node.

### Why it matters

If anything feels wrong later, you can instantly revert.

### Commands (MiniBolt)

**Stop services**
```bash
# LND
sudo systemctl stop lnd

# Nextcloud SCB backup (custom service/timer) / UNIQUE TO ME
sudo systemctl stop nextcloud-scb-upload.timer || true
sudo systemctl stop nextcloud-scb-upload.service || true
sudo systemctl disable nextcloud-scb-upload.timer

# Local and Github
sudo systemctl stop scb-backup.service || true

#Confirm no timers
systemctl list-timers | grep scb
systemctl status nextcloud-scb-upload.service


```

**Create rollback**
```
sudo mkdir -p /data/lnd.minibolt-init-backup
sudo rsync -av /data/lnd/ /data/lnd.minibolt-init-backup/
```

### Success looks like

* MiniBolt LND fully stopped
* Initial state captured
* No automation running

---

## 5. Phase 3 — Data Transplant (Frozen Snapshot)

### What this does

Copies the authoritative LND state exactly once.

### Why it matters

This is the actual migration of identity, channels, and funds.

### Commands (run on Raspiblitz)

```bash
sudo rsync -av --progress \
  /mnt/hdd/lnd.FROZEN-* /mnt/hdd/lnd.FROZEN-CURRENT
sudo rsync -av --progress \
  /mnt/hdd/lnd.FROZEN-CURRENT/ \
  admin@minibolt:/data/lnd/
```

*(Replace /mnt/hdd/lnd.FROZEN-YYYYMMDD with your actual frozen directory name.)*

### Fix ownership (MiniBolt)

```bash
sudo chown -R lnd:lnd /data/lnd
sudo chmod -R 700 /data/lnd
```

### Validation

```bash
sudo ls -lh /data/lnd/data/chain/bitcoin/mainnet
sudo ls -l /data/lnd/v3_onion_private_key
```

You should now see:

* `wallet.db` (old size)
* `channel.backup` (~old size)
* ownership `lnd:lnd`
* /data/lnd/v3_onion_private_key present and owned by lnd:lnd

⚠️ Do not start LND on the target node before copying `v3_onion_private_key`,
or a new Tor identity will be generated.

---

## 6. Phase 4 — Configuration Reconciliation (DO NOT RUSH)

### What this does

Merges old node behavior with MiniBolt wiring.

### Why it matters

Most migration failures happen here, not in the copy.

### Review these together

```bash
sed 's/^/    /' /data/lnd/lnd.conf | less
sudo systemctl cat lnd
cat /data/bitcoin/bitcoin.conf
```

### Rules

* ✅ Use MiniBolt `bitcoind.rpc*` + `zmq*`
* ✅ Keep old node behavioral options intentionally
* ❌ No references to `/mnt/hdd`
* ❌ No Postgres options

Do **not** start LND until this is correct.

---

## 7. Phase 5 — First Start (DB Migration)

### What this does

Starts LND exactly once with the migrated data.

### Why it matters

This is where DB upgrades happen.

### Commands

```bash
sudo systemctl start lnd
journalctl -fu lnd
```

Then unlock:

```bash
sudo -u lnd lncli unlock
```

### Expected logs

* DB migration messages
* Channel recovery logs
* Normal peer reconnect attempts

🚨 **STOP if you see**

* DB corruption errors
* Continuous restart loops
* Permission errors

---

## 8. Phase 6 — Manual Validation (NO AUTOMATION YET)

### Identity

```bash
lncli getinfo
```

✅ `identity_pubkey` matches old node
✅ `uris` contains the same .onion address as before

### Channels

```bash
lncli listchannels
lncli pendingchannels
```

✅ Channels appear / recovering

### Balances

```bash
lncli walletbalance
lncli channelbalance
```

✅ Totals look sane

### Logs

```bash
journalctl -u lnd -n 100 --no-pager
```

✅ Logs boring and stable

---

## 9. Phase 7 — Cooling Period

### What this does

Lets the node settle without interference.

### Why it matters

Late issues show up here, not at first start.

**Recommendation**

* Let LND run quietly for several hours (or overnight)
* No UIs
* No backup automation

---

## 10. Phase 8 — Re-enable Automation

### SCB backups

```bash
sudo systemctl enable --now nextcloud-scb-upload.timer
```

Verify:

```bash
journalctl -u nextcloud-scb-upload
ls -lh /data/lnd/remote-lnd-backup
sudo systemctl start scb-backup.service
```

### Optional UIs

```bash
sudo systemctl start rtl
sudo systemctl start thunderhub
```

---

## 11. Phase 9 — Final Commit

### What this does

Declares MiniBolt the production node.

### Actions

* Export a new SCB
* Back it up off-box
* Keep Raspiblitz powered off permanently
* Retain `/mnt/hdd/lnd.FROZEN-*` as cold archive

---

## 12. Abort & Rollback Logic

If you abort **before Phase 7**:

```bash
sudo systemctl stop lnd
sudo rm -rf /data/lnd
sudo mv /data/lnd.minibolt-init-backup /data/lnd
sudo chown -R lnd:lnd /data/lnd
```

Old node remains frozen unless you explicitly restore it.

