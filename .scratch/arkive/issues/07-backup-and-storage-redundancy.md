# Backup Automation & Storage Redundancy Interface

Type: grilling
Status: resolved
Blocked by: 04

## Question

How should Arkive interface with underlying filesystem topologies (e.g. single drive vs. SnapRAID + mergerfs vs. Linux mdadm RAID1) and automate offsite cloud backups without thrashing drives or exhausting system resources during active hours?

Specific decisions to resolve with the user:
1. **Application vs. Host Boundary:** Should Arkive attempt to manage disk arrays (mergerfs/SnapRAID) directly, or simply accept a target filesystem mount path (`/mnt/storage`) and expose system health/monitoring status (SMART data, available space, spin status)?
2. **Backup Integration Hook:** How should offsite backups be triggered? (e.g., Arkive providing pre-backup database dump/snapshot hooks for external tools like `rclone` / `restic`, or Arkive embedding a lightweight scheduler to invoke `rclone sync` during low-traffic overnight hours)?
3. **Backup Scope & Priority:** Does offsite sync back up only database metadata + raw media on `/mnt/storage`, safely omitting the ephemeral preview cache on SSD? How should the backup process be throttled (`ionice -c3` / bandwidth limits) so it does not interfere with daytime media viewing?

## Answer

**Verdict: Decouple hardware storage topologies to the host OS, automate atomic overnight database dumps with a 7-day rolling retention and configurable offsite sync hook, exclude ephemeral SSD preview caches from backup scopes, throttle execution with idle I/O priority, and provide an automated `arkive cache rebuild` routine for disaster recovery.**

### Agreed Decisions
1. **Host-Level Storage Boundary & Non-Waking Telemetry**:
   - Arkive operates strictly against standard filesystem mount paths (`/mnt/storage` for raw media and `/var/cache/arkive/previews` for the SSD preview cache).
   - Array redundancy (mdadm), parity (SnapRAID), and pooling (mergerfs) remain host OS concerns, keeping Docker containers unprivileged.
   - Filesystem capacity metrics (total, used, free percentage) are read via non-waking `statvfs` syscalls and displayed in the Admin panel without waking spun-down drives.
2. **Automated Overnight Database Snapshots & Rolling Retention**:
   - An Asynq cron worker executes off-peak (03:00 AM) to write an atomic database dump to `/mnt/storage/backups/`.
   - A 7-day rolling retention window purges dumps older than 7 days, maintaining a week of historical point-in-time restore targets while bounding local disk usage.
   - An optional offsite sync command (such as `rclone sync` or a custom shell hook) is triggered immediately following the database export.
3. **Backup Scope & Ephemeral Cache Exclusion**:
   - Backup target includes only the database snapshot and the Raw Media Store (`/mnt/storage/library/`).
   - The SSD Preview Cache (`/var/cache/arkive/previews/`), temporary upload staging files (`.staging/`), and Valkey session states are omitted, saving cloud storage costs and backup transmission time.
4. **Resource Throttling & Network Bandwidth Caps**:
   - Backup execution is restricted to low CPU priority (`nice -n 19`) and idle I/O (`ionice -c 3`), yielding instantly to interactive user requests.
   - An optional bandwidth limit (`BACKUP_BANDWIDTH_LIMIT_KBPS`) prevents offsite uploads from saturating residential internet upstream connections.
5. **Disaster Recovery & Preview Cache Rebuilding**:
   - When restoring from backup (where database metadata and original media are present but the SSD Preview Cache is empty), administrators can execute `arkive cache rebuild` via CLI or trigger "Rebuild Previews" in the Admin dashboard.
   - The rebuild engine scans the database for missing previews and queues generation jobs into the throttled background worker queue on Core 1, smoothly repopulating the SSD cache without degrading system responsiveness.

