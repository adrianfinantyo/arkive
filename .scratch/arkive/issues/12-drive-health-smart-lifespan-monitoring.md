# Hard Drive Health & Lifespan Monitoring (S.M.A.R.T.)

Type: grilling
Status: resolved
Blocked by: 04, 07

## Question

How should Arkive collect, evaluate, and display hard drive health metrics, operational duration (power-on hours), and estimated remaining lifespan using S.M.A.R.T. telemetry without waking the mechanical drive when it is spun down?

Specific decisions to resolve with the user:
1. **Drive Spin-Down Preservation during Telemetry:** How does Arkive query S.M.A.R.T. status without waking a sleeping drive (e.g. `smartctl -n standby`), and how should telemetry metrics be cached in SQLite/Valkey when the drive is active?
2. **Lifespan & Wear Estimation Formula:** Which primary S.M.A.R.T. attributes (e.g. Power-On Hours ID 9, Reallocated Sector Count ID 5, Current Pending Sectors ID 197, Load Cycle Count ID 193 for HDD; TBW / Percentage Used for SSD) should determine the health percentage and estimated lifespan display?
3. **Container Permissions & Host Interface:** How should the containerized Go service access block device telemetry? (e.g., exposing `/dev/sdX` with `CAP_SYS_RAWIO` vs reading a host cron JSON export from `/mnt/storage/.smart/`)?
4. **Admin Dashboard Visuals & Warning UX:** How should drive health, duration used (e.g. "Used for 4 years, 2 months"), temperature, and early drive replacement warnings be surfaced to non-technical users in the Admin panel?

## Answer

**Verdict: Preserve drive spin-down by strictly checking power mode before querying S.M.A.R.T. telemetry, isolate host storage privileges via a non-root sidecar or host cron dumping to a shared state path on SSD, evaluate health through critical sector thresholds and power-on operational age, and surface status in the Admin dashboard using plain-English health indicators and sleeping badges.**

### Agreed Decisions
1. **Standby-Preserving Telemetry & SSD Cache Tier**:
   - Query drives using `smartctl -a -j -n standby` (ATA `CHECK POWER MODE`).
   - If the mechanical storage drive is sleeping, `smartctl` exits with code 2 without waking platters.
   - Serve last-known metrics from Valkey and PostgreSQL on the root SSD, displaying a "Standby (Sleeping)" badge.
   - Telemetry refreshes opportunistically when the drive is already awake during media uploads or the 03:00 AM database backup worker.
2. **Deterministic Health Tiers & Operational Age**:
   - Evaluate health using three deterministic states:
     - *Healthy (Green):* Zero reallocated (ID 5), pending (ID 197), or uncorrectable (ID 198) sectors; normal operating temperatures (<50°C HDD, <60°C SSD).
     - *Warning (Yellow):* Reallocated sectors 1 to 49, load cycles >= 300,000, temperature >= 50°C, or SSD write wear >= 80%.
     - *Critical (Red):* Active pending sectors (ID 197 > 0), uncorrectable sectors (ID 198 > 0), reallocations >= 50, or SSD wear >= 100%.
   - Compute human operational age from raw power-on hours: "Used for 3 years, 7 months".
3. **Decoupled Host Exporter & Unprivileged Containers**:
   - Application containers remain unprivileged per ADR-0009; no `/dev/sdX` device mounts or `CAP_SYS_RAWIO` capabilities are granted.
   - Host utility or sidecar targets persistent block device paths (`/dev/disk/by-id/*`) and writes an atomic JSON snapshot to the root SSD (`/var/run/arkive/smart.json`).
   - The state file is mounted read-only into application containers. Telemetry files are never written to `/mnt/storage`.
4. **Admin UX with Plain-English Actionable Guidance**:
   - Primary dashboard cards display status badges, sleeping state indicators, operational age, temperature gauges, and non-waking capacity metrics from `statvfs`.
   - Warnings surface plain-English explanations with direct backup triggers (`[Run Backup Now]`).
   - Technical S.M.A.R.T. tables are kept inside a collapsible drawer for advanced inspection.
