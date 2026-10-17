# Hardware Telemetry Alerting & Webhooks

Type: grilling
Status: open
Blocked by: 07, 12

## Question

How should Arkive evaluate critical hardware status (S.M.A.R.T. health, drive overheating, storage exhaustion, backup failures) and dispatch push notifications to homelab administrators without requiring outbound SMTP email servers?

Specific decisions to resolve with the user:
1. **Notification Channels & Providers:** Which self-hosted and homelab-standard notification services should Arkive support natively (e.g. Telegram Bot API, Gotify, ntfy, Pushover, generic webhook)?
2. **Alert Triggers & Severity Thresholds:** Which system events trigger alerts (e.g. HDD pending sectors > 0, disk temperature >= 50°C, storage capacity > 90%, overnight database backup failure)?
3. **Deduplication & Rate Limiting:** How should the alerting worker prevent notification storms when a metric remains in a warning or critical state for consecutive days?
4. **Evaluation Engine & Worker Scheduling:** How does the Asynq worker daemon evaluate alerts asynchronously without waking spun-down drives or blocking interactive web requests?
