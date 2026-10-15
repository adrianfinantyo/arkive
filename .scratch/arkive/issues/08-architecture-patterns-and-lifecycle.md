# Architecture Patterns, Code Structure & Data Lifecycle

Type: grilling
Status: claimed
Blocked by: 01

## Question

What architecture patterns, package structure, database access layers, and data lifecycle policies should Arkive adopt to match the developer's engineering preferences while maintaining strict hardware constraints (<100MB RAM, zero Redis bloat, low-latency Go runtime)?

Specific decisions to resolve with the user:
1. **Application Layering & Architecture Pattern:** Should Arkive use a pragmatic layered package structure (`cmd/`, `internal/api/`, `internal/domain/`, `internal/store/`, `internal/media/`, `internal/worker/`) or strict hexagonal / clean architecture (ports & adapters)?
2. **Database Access Strategy:** Should Arkive use `sqlc` (type-safe Go code generated from raw SQL queries), raw SQL with lightweight scanning helpers, or a lightweight query builder?
3. **In-Memory Caching & Session State (Zero Redis):** Without running a heavy external Redis daemon, how should short-lived state (auth sessions, rate limiting, temporary link tokens) be handled? (e.g. in-process thread-safe Go memory cache + SQLite persistence vs. purely database-backed sessions)?
4. **Trash Bin Lifecycle & Idempotent Permanent Removal:** How should the soft-delete trash bin operate? Specifically: retention period (default 30 days vs configurable), manual "Empty Trash" vs automated background cron, and atomic cascade deleting the DB record, raw file on `/mnt/storage`, and WebP previews on SSD.
