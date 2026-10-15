# Architecture Patterns, Code Structure & Data Lifecycle

Type: grilling
Status: resolved
Blocked by: 01

## Question

What architecture patterns, package structure, database access layers, and data lifecycle policies should Arkive adopt to match the developer's engineering preferences while maintaining strict hardware constraints (<100MB RAM, zero Redis bloat, low-latency Go runtime)?

Specific decisions to resolve with the user:
1. **Application Layering & Architecture Pattern:** Should Arkive use a pragmatic layered package structure (`cmd/`, `internal/api/`, `internal/domain/`, `internal/store/`, `internal/media/`, `internal/worker/`) or strict hexagonal / clean architecture (ports & adapters)?
2. **Database Access Strategy:** Should Arkive use `sqlc` (type-safe Go code generated from raw SQL queries), raw SQL with lightweight scanning helpers, or a lightweight query builder?
3. **In-Memory Caching & Session State (Zero Redis):** Without running a heavy external Redis daemon, how should short-lived state (auth sessions, rate limiting, temporary link tokens) be handled? (e.g. in-process thread-safe Go memory cache + SQLite persistence vs. purely database-backed sessions)?
4. **Trash Bin Lifecycle & Idempotent Permanent Removal:** How should the soft-delete trash bin operate? Specifically: retention period (default 30 days vs configurable), manual "Empty Trash" vs automated background cron, and atomic cascade deleting the DB record, raw file on `/mnt/storage`, and WebP previews on SSD.

## Answer

**Verdict: Adopt an Open-Closed Principle (OCP) layered monorepo with GORM persistence abstraction, Valkey caching, Asynq background workers, a togglable two-container deployment (`arkive-server` + `arkive-worker`), and an automated $X$-day trash purge lifecycle.**

### Agreed Architectural Decisions
1. **Monorepo Layout & Open-Closed Principle (OCP)**:
   - A single repository holding the Go backend and SvelteKit frontend.
   - Core domain models and business contracts live in `internal/domain/` as clear interfaces (`MediaRepository`, `UserRepository`, `ShareLinkRepository`, `StorageBackend`).
   - Domain logic depends strictly on interfaces, not concrete ORM or driver structs, allowing new features or backends to be added without modifying existing domain rules.
2. **Database Layer (ORM for Dialect Portability)**:
   - Adopt GORM (`gorm.io/gorm`) implementing the domain repository interfaces.
   - Enables seamless switching between PostgreSQL, SQLite, or MySQL with zero business logic rewrites.
   - Built-in `gorm.DeletedAt` handles soft-delete state for the Trash Bin natively.
3. **Cache & Worker Subsystems**:
   - **Valkey 8 (`valkey/valkey:8-alpine`)**: Used for session caching, token rate limiting (`valkey-go`), and task queue backing.
   - **`hibiken/asynq`**: Manages asynchronous media jobs, retry backoffs, worker bounding, and scheduled daily cron tasks.
4. **Two-Container Split Deployment**:
   - `arkive-server` (`cmd/server`, `Dockerfile.server`): Lightweight (~25MB) web/API/streaming container without media CLI dependencies.
   - `arkive-worker` (`cmd/worker`, `Dockerfile.worker`): Background worker container with `vipsthumbnail` + `ffmpeg`. Togglable via Docker Compose profile (`profiles: ["worker"]`) and isolated to 1 CPU core and 384MB RAM.
5. **Trash Bin Lifecycle**:
   - Deleting files/albums sets `deleted_at = NOW()`.
   - Items remain in the user's Trash view with an option to restore.
   - A daily `asynq` cron worker permanently purges items older than $X$ days (default 30), atomically deleting the DB record, the raw file on `/mnt/storage`, and the WebP preview files on the SSD.
