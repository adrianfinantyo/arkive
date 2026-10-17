## Destination

A complete technical specification and architecture blueprint for **Arkive**, detailing the Go backend runtime, direct-to-disk streaming upload pipeline, database schema (GORM with PostgreSQL/SQLite portability), Valkey cache and Asynq worker queue, two-tier SSD/HDD storage engine with drive spin-down protection, embedded SvelteKit PWA, dynamic permission/sharing engine, and backup/redundancy integration: fully prepared for phased implementation on low-spec hardware (Intel Pentium E5400, 3.8GB RAM).

## Notes

- Domain: Media archiving, self-hosted file management, low-power homelab infrastructure.
- Consult skills: `grilling`, `domain-modeling`, `codebase-design`.
- Read vocabulary: [`GLOSSARY.md`](file:///home/adrian/adrian-dev/arkive/GLOSSARY.md).
- Hardware constraints: Intel Pentium Dual-Core E5400 (2C/2T @ 2.7GHz), 3.8 GiB RAM, 256GB SSD (`/`), 1TB WD HDD (`/mnt/storage`).
- Key invariants: Docker Compose stack tuned for low idle RAM (~120–150MB total); HDD spin-down preserved during browsing/galleries; streaming uploads bypass RAM buffering; no office suites, desktop sync daemons, or machine-learning indexing.

## Decisions so far

<!-- the index: one line per closed ticket, enough to judge relevance, then zoom the link for the detail the ticket holds -->

- [[01] Backend Runtime & Framework Evaluation](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/01-backend-runtime-evaluation.md): Reject PocketBase due to 32MB upload RAM buffering, SSD /tmp spillover, and WAL telemetry that defeats HDD spin-down; build a bespoke Go service using Chi v5 for an 8–14MB idle footprint and 0-RAM streaming direct to `/mnt/storage` (database layer evolved to GORM + PostgreSQL via ADR-0004/0005).
- [[02] Low-Resource Media Pipeline](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/02-low-resource-media-pipeline.md): Adopt isolated CLI subprocesses (`vipsthumbnail` + fast-seek `ffmpeg`) bound to Core 1 (`nice -n 19`, `ionice -c 3`, `prlimit 512MB`) with eager-on-upload processing to keep Go heap <25MB and protect HDD spin-down.
- [[03] Frontend Architecture & Single-Binary PWA Delivery](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/03-frontend-pwa-architecture.md): Embed SvelteKit SPA into Go binary via `//go:embed all:build` with build-time precompression; sustain 60fps mobile gallery scrolling with $O(1)$ uniform-row Svelte 5 DOM virtualization (~40 DOM nodes, <25MB RAM); enforce 500-entry FIFO client Service Worker Preview Cache to insulate HDD spin-down without storage exhaustion; load PhotoSwipe v5 on-demand for touch gestures and inline video.
- [[04] Storage Tiering & Drive Spin-Down Isolation](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/04-storage-tiering-spindown-isolation.md): Maintain HDD spin-down via eager upload thumbnailing to SSD, full metadata isolation in the database on SSD (forbidding storage drive stat/readdir during browsing), warm consumer UX ("Preparing media..."), and synchronized thumbnail lifecycle with trash bin auto-purge.
- [[05] Dynamic Sharing & Permission Engine Schema](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/05-dynamic-sharing-permission-schema.md): Model direct column visibility (private, group, public) with indexed Materialized Paths on folders, three-tier roles (Admin, Uploader, Viewer) with Group sharing, eager dual-format previews (WebP for gallery, JPEG for universal downloads) on SSD, and resource-bounded video streaming.
- [[06] Streaming Upload & Buffer Control Architecture](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/06-streaming-upload-buffering.md): Implement tus resumable uploads streaming directly to `/mnt/storage/.staging/` with pooled 64KB buffers, a 2-writer disk semaphore with round-robin chunk queueing, `fallocate()` contiguous block pre-allocation, and read-prioritized 5-stream video governance.
- [[07] Backup Automation & Storage Redundancy Interface](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/07-backup-and-storage-redundancy.md): Decouple hardware storage topologies to the host OS, automate atomic overnight database dumps with a 7-day rolling retention and configurable offsite sync hook, exclude ephemeral SSD preview caches from backup scopes, throttle execution with idle I/O priority, and provide an automated `arkive cache rebuild` routine for disaster recovery.
- [[08] Architecture Patterns, Code Structure & Data Lifecycle](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/08-architecture-patterns-and-lifecycle.md): Adopt an Open-Closed Principle (OCP) layered monorepo with GORM persistence abstraction across DB engines, Valkey caching, Asynq background workers, a togglable two-container deployment (`arkive-server` + `arkive-worker`), and an automated $X$-day trash purge lifecycle.

## Not yet specified

- **On-the-fly Streaming Zip Archives**: Zero-temp-file streaming zip generator for batch downloads directly from HDD to HTTP response body.
- **REST API Specification**: Contract definition between Go backend and SvelteKit frontend for galleries, albums, auth, and link sharing.
- **User Invitation & Account Recovery**: Low-overhead onboarding flow for non-technical community users without complex SMTP requirements.

## Out of scope

- Collaborative online document editing (OnlyOffice, Collabora, WOPI integrations).
- Heavy background desktop sync daemons with two-way file conflict resolution.
- AI/ML features (facial recognition, object classification, vector embedding search).
- Multi-node clustering or distributed consensus algorithms (Arkive is single-node/homelab-first).
