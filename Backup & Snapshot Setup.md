## Backup & Snapshot Setup (Proxmox)

### Snapshots (quick rollback points)

Snapshots capture the VM's disk (and optionally RAM) state instantly — useful before risky changes, not a substitute for backups.

**Steps:**
1. Select the VM in the Proxmox tree
2. Click **Snapshots** in the left panel
3. Click **Take Snapshot**
4. Name it clearly (e.g. `post-install-working`, `clean-2026-07-28`)
5. Optionally check **Include RAM** to capture the exact running state
6. Click **Create**

**To roll back:**
- Snapshots tab → select the snapshot → click **Rollback**

> Note: snapshots live on the same storage as the VM disk — don't rely on them as your only protection, and don't let them pile up indefinitely.

---

### Backup Jobs (real, independent copies)

**1. Create a backup job**
- Datacenter → **Backup** → **Add**
- **Node**: select node or "All"
- **Storage**: choose a dedicated backup storage target (not the OS root disk — see note below)
- **Schedule**: e.g. daily at 01:00
- **Selection mode**: "Include selected VMs" → check the VMs to back up
- **Mode**: `Snapshot` (no downtime)
- **Compression**: `zstd`
- Click **Create**

**2. Set retention** (Storage → your backup storage → Edit → Backup Retention tab)

| Setting | Example |
|---|---|
| Keep Last | 3 |
| Keep Daily | 7 |
| Keep Weekly | 4 |
| Keep Monthly | 1 |
| Keep Yearly | 1 |

**3. Test immediately**
- Open the job → **Run now**
- Confirm it completes without errors before relying on the schedule

**4. Verify backups are landing correctly**
- Storage → backup storage → **Content** tab — confirms each VM's backup file, date, and size

**5. Set up notifications**
- Datacenter → **Notifications** → configure a target (e.g. sendmail) so job failures alert you automatically instead of going unnoticed

---

### ⚠️ Critical lesson: don't put backup storage on the OS root disk

Backup storage pointed at `/home/...` on the Proxmox root filesystem (`pve-root`) filled to 100% and broke all backups host-wide. Fix:

1. Create a directory on a large dedicated volume:
```bash
   mkdir -p /mnt/storage1/main-backup
```
2. Move existing backup files there:
```bash
   mv /home/main-backup/dump/* /mnt/storage1/main-backup/
```
3. Remove the old storage definition (Datacenter → Storage → select it → **Remove** — this only removes the Proxmox reference, not the files)
4. Add a new storage entry:
   - **Add → Directory**
   - **ID**: same name as before (so existing backup jobs don't need reconfiguring)
   - **Directory**: `/mnt/storage1/main-backup`
   - **Content**: `Disk image`, `VZDump backup file`
5. Confirm the backup job's Storage field still points to the correct ID, then **Run now** to verify

**Also note:** retention policies only prune *after* a successful backup — if backups keep failing (e.g. due to full disk), old backups silently accumulate instead of being cleaned up.
