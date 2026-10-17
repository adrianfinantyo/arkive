# Hard Drive Health & Lifespan Monitoring (S.M.A.R.T.)

Type: grilling
Status: open
Blocked by: 04, 07

## Question

How should Arkive collect, evaluate, and display hard drive health metrics, operational duration (power-on hours), and estimated remaining lifespan using S.M.A.R.T. telemetry without waking the mechanical drive when it is spun down?

Specific decisions to resolve with the user:
1. **Drive Spin-Down Preservation during Telemetry:** How does Arkive query S.M.A.R.T. status without waking a sleeping drive (e.g. `smartctl -n standby`), and how should telemetry metrics be cached in SQLite/Valkey when the drive is active?
2. **Lifespan & Wear Estimation Formula:** Which primary S.M.A.R.T. attributes (e.g. Power-On Hours ID 9, Reallocated Sector Count ID 5, Current Pending Sectors ID 197, Load Cycle Count ID 193 for HDD; TBW / Percentage Used for SSD) should determine the health percentage and estimated lifespan display?
3. **Container Permissions & Host Interface:** How should the containerized Go service access block device telemetry? (e.g., exposing `/dev/sdX` with `CAP_SYS_RAWIO` vs reading a host cron JSON export from `/mnt/storage/.smart/`)?
4. **Admin Dashboard Visuals & Warning UX:** How should drive health, duration used (e.g. "Used for 4 years, 2 months"), temperature, and early drive replacement warnings be surfaced to non-technical users in the Admin panel?
