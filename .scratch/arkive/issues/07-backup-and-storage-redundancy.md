# Backup Automation & Storage Redundancy Interface

Type: grilling
Status: open
Blocked by: 04

## Question

How should Arkive interface with underlying filesystem topologies (e.g. single drive vs. SnapRAID + mergerfs vs. Linux mdadm RAID1) and automate offsite cloud backups without thrashing drives or exhausting system resources during active hours?

Specific decisions to resolve with the user:
1. **Application vs. Host Boundary:** Should Arkive attempt to manage disk arrays (mergerfs/SnapRAID) directly, or simply accept a target filesystem mount path (`/mnt/storage`) and expose system health/monitoring status (SMART data, available space, spin status)?
2. **Backup Integration Hook:** How should offsite backups be triggered? (e.g., Arkive providing pre-backup SQLite snapshot hooks for external tools like `rclone` / `restic`, or Arkive embedding a lightweight cron-like scheduler to invoke `rclone sync` during low-traffic overnight hours)?
3. **Backup Scope & Priority:** Does offsite sync back up only SQLite database metadata + raw media on `/mnt/storage`, safely omitting the ephemeral preview cache on SSD? How should the backup process be throttled (`ionice -c3` / bandwidth limits) so it does not interfere with daytime media viewing?
