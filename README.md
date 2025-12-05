# ⚡ LND Migration Guide (Raspiblitz, Bare-metal / MiniBolt)

# ⚠️ NOT YET TESTED ⚠️

---

> > This document records the exact LND migration process I plan to use.
> It is shared for reference and feedback, not as a universal guide.
> Your paths, firewall rules, and `lnd.conf` will likely differ.

## Assumptions & Rules (READ FIRST)

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

## Version Handling (Important Context)

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

## Phase 0 — Pre-Flight Checks (No Risk, Offline)

### What this does

Confirms data, config, and environment are sane *before* touching anything.

### Why it matters

Catches wiring errors early and safely.

### Commands

**Emergency Contact Info**
This information is not used during a normal migration, but ensures you know where recovery materials are before proceeding.
```
## 🆘 Emergency Recovery
- **Seed phrase location**: [YOUR_SECURE_LOCATION]
- **Latest SCB backup**: [YOUR_BACKUP_LOCATION]  
```

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

**Check for pending HTLCs or force-closing channels**
```
# On Raspiblitz - Check for any pending HTLCs or force-closing channels
sudo -u bitcoin lncli listchannels | grep -E "(pending_htlcs|waiting_close)"
sudo -u bitcoin lncli pendingchannels

```


---

## Phase 1 — Freeze Old Node (Hard Lock)

### What this does

Permanently prevents the old node from starting LND accidentally.

### Why it matters

This removes *all* dual-node risk, even from hidden services or human error.

### Commands (Raspiblitz)

```bash
# Stop all related services first
sudo systemctl stop lnd rtl thunderhub
sudo systemctl disable lnd rtl thunderhub

# Export Channel backup
sudo -u bitcoin lncli exportchanbackup --all

# Freeze
sudo mv /mnt/hdd/lnd /mnt/hdd/lnd.FROZEN-$(date +%Y%m%d)

# NEW: Generate checksums of critical database files
sudo find /mnt/hdd/lnd.FROZEN-* -name "*.db" -exec sha256sum {} \; > /tmp/raspiblitz_db_checksums.txt
sudo cp /tmp/raspiblitz_db_checksums.txt /home/admin/
echo "Database checksums saved to /home/admin/raspiblitz_db_checksums.txt"

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

## Phase 2 — Prepare MiniBolt (Rollback Safety)

### What this does

Creates a full rollback point on the new node.

### Why it matters

If anything feels wrong later, you can instantly revert.

### Commands (MiniBolt)

**Stop services**
```bash
# LND
sudo systemctl stop lnd

# Custom service/timers (Examples: Nextcloud, rsync jobs, cron, cloud sync)
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
## ✅ Migration Blackout Entry Check (REQUIRED)

Before proceeding, confirm **both nodes are inert**.

### Old Node — Raspiblitz

- `/mnt/hdd/lnd` has been renamed to `/mnt/hdd/lnd.FROZEN-*`
- `/mnt/hdd/lnd` no longer exists
- `systemctl status lnd` shows inactive
- Final SCB exported and timestamp verified
- Current `.onion` URI noted from `lncli getinfo` (for later comparison)

### New Node — MiniBolt

- `lnd` is stopped
- All SCB automation stopped and disabled
- RTL / Thunderhub / any UI stopped
- No timers touching `/data/lnd`
- `/data/lnd.minibolt-init-backup` created

✅ **Do not proceed to data copy unless ALL items above are confirmed.**

---

## Phase 3 — Data Transplant (Frozen Snapshot)

### What this does

Copies the authoritative LND state exactly once.

### Why it matters

This is the actual migration of identity, channels, and funds.

### Commands (run on Raspiblitz)

```bash
# TIMING: Aim to complete Phases 3-5 within 2 hours to minimize peer timeout risk
echo "Migration started at: $(date)"

sudo rsync -av --progress \
  /mnt/hdd/lnd.FROZEN-* /mnt/hdd/lnd.FROZEN-CURRENT
sudo rsync -av --progress \
  /mnt/hdd/lnd.FROZEN-CURRENT/ \
  admin@minibolt:/data/lnd/

```

*(Replace /mnt/hdd/lnd.FROZEN-CURRENT with your actual frozen directory name.)*

### Copy checksum file for verification 
```
scp /home/admin/raspiblitz_db_checksums.txt admin@minibolt:/tmp/
```


### Fix ownership (MiniBolt)

```bash
sudo chown -R lnd:lnd /data/lnd
sudo chmod -R 700 /data/lnd

```

### Verify data integrity (Advanced / Optional)
```
sudo find /data/lnd -name "*.db" -exec sha256sum {} \; > /tmp/minibolt_db_checksums.txt
echo "=== DATABASE INTEGRITY CHECK ==="
diff /tmp/raspiblitz_db_checksums.txt /tmp/minibolt_db_checksums.txt
if [ $? -eq 0 ]; then
    echo "✅ Database checksums match - copy successful"
else
    echo "❌ Database checksums differ - STOP and investigate"
    exit 1
fi
```

> ⚠️ **Important**
> These checksums confirm **filesystem copy integrity only** (no corruption or partial transfer).
> They do **not** guarantee Lightning protocol safety, channel state correctness,
> or successful recovery after LND startup.


### Validation

```bash
sudo ls -lh /data/lnd/data/chain/bitcoin/mainnet
sudo ls -l /data/lnd/v3_onion_private_key

# Verify TLS files are copied
sudo ls -l /data/lnd/tls.cert /data/lnd/tls.key


```

You should now see:

* `wallet.db` (old size)
* `channel.backup` (~old size)
* ownership `lnd:lnd`
* /data/lnd/v3_onion_private_key present and owned by lnd:lnd

⚠️ Do not start LND on the target node before copying `v3_onion_private_key`,
or a new Tor identity will be generated.

---

## Phase 4 — Configuration Reconciliation (DO NOT RUSH)

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

### Verify Paths
```
# Verify critical paths match
grep -E "(bitcoind\.|zmq)" /data/lnd/lnd.conf
grep -E "(datadir|rpcuser|rpcpassword)" /data/bitcoin/bitcoin.conf

# Ensure no Raspiblitz-specific paths remain
grep -r "/mnt/hdd" /data/lnd/ || echo "No old paths found - good!"
```

### Ensure Network Connectivity
```
# Ensure Bitcoin Core connectivity
bitcoin-cli getblockchaininfo
bitcoin-cli getnetworkinfo
```

### Rules

* ✅ Use MiniBolt `bitcoind.rpc*` + `zmq*`
* ✅ Keep old node behavioral options intentionally
* ❌ No references to `/mnt/hdd`
* ❌ No Postgres options

Do **not** start LND until this is correct.

---

## ✅ Pre-Start Final Sanity Check

Before starting LND for the first time on MiniBolt:

- No LND processes are running anywhere else
- Old node remains frozen and powered off
- `/data/lnd` contains migrated data only
- `v3_onion_private_key` is present and owned by `lnd:lnd`
- No automation or UI processes are enabled
- You are prepared for forward-only DB migration

✅ **Once LND starts, Lightning network interactions may occur.**


## ⚠️⚠️ Point of no return ⚠️⚠️
---

## Phase 5 — First Start (DB Migration)

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

## Phase 6 — Manual Validation (NO AUTOMATION YET)

### Identity

```bash
sudo -u lnd lncli getinfo

# Compare with noted address from old node
sudo -u lnd lncli getinfo | grep uris

```

✅ `identity_pubkey` matches old node
✅ `uris` contains the same .onion address as before

### Peers

```bash
# Check peer connections after startup
sudo -u lnd lncli listpeers
sudo -u lnd lncli getnetworkinfo
```

✅ Peers are connecting


### Channels

```bash
sudo -u lnd lncli listchannels
sudo -u lnd lncli pendingchannels
```

✅ Channels appear / recovering

### Macaroon
```
sudo ls -l /data/lnd/data/chain/bitcoin/mainnet/*.macaroon

```

✅ Macaroon present
### Balances

```bash
sudo -u lnd lncli walletbalance
sudo -u lnd lncli channelbalance
```

✅ Totals look sane

### Logs

```bash
journalctl -u lnd -n 100 --no-pager
```

✅ Logs boring and stable

---

## Phase 7 — Cooling Period

### What this does

Lets the node settle without interference for 24 hours.

### Why it matters

Late issues show up here, not at first start.

**Recommendation**

* Let LND run quietly for several hours (or overnight)
* No UIs
* No backup automation

---

## Phase 8 — Re-enable Automation

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

## Phase 9 — Final Commit
### What this does
Declares MiniBolt the production node.

### Final Success Validation
```
# Verify all expected channels are active (not just present)
sudo -u lnd lncli listchannels --active_only | wc -l
# Should match your expected active channel count

# Verify no force-closed channels
sudo -u lnd lncli pendingchannels | grep -c "waiting_close_channels" || echo "0"
# Should be 0
```

* Export a new SCB
* Back it up off-box
* Keep Raspiblitz powered off permanently
* Retain `/mnt/hdd/lnd.FROZEN-*` as cold archive

---

## Abort & Rollback Logic (Minibolt Only)
If you decide not to proceed any further (at any point up to and including Phase 6):

This rollback restores the MiniBolt node’s local filesystem state.
It does **not** undo Lightning network interactions that may have
occurred after first start.


```bash
sudo systemctl stop lnd
sudo rm -rf /data/lnd
sudo mv /data/lnd.minibolt-init-backup /data/lnd
sudo chown -R lnd:lnd /data/lnd
```
