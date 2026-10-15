## Destination

A complete technical specification and architecture blueprint for **Arkive**, detailing the Go backend runtime, direct-to-disk streaming upload pipeline, SQLite schema, two-tier SSD/HDD storage engine with drive spin-down protection, embedded SvelteKit PWA, dynamic permission/sharing engine, and backup/redundancy integration—fully prepared for phased implementation on low-spec hardware (<100MB idle RAM, Intel Pentium E5400).

## Notes

- Domain: Media archiving, self-hosted file management, low-power homelab infrastructure.
- Consult skills: `grilling`, `domain-modeling`, `codebase-design`.
- Read vocabulary: [`GLOSSARY.md`](file:///home/adrian/adrian-dev/arkive/GLOSSARY.md).
- Hardware constraints: Intel Pentium Dual-Core E5400 (2C/2T @ 2.7GHz), 3.8 GiB RAM, 256GB SSD (`/`), 1TB WD HDD (`/mnt/storage`).
- Key invariants: Idle RAM < 100MB; HDD spin-down preserved during browsing/galleries; streaming uploads bypass RAM buffering; no office suites, desktop sync daemons, or machine-learning indexing.

## Decisions so far

<!-- the index: one line per closed ticket, enough to judge relevance, then zoom the link for the detail the ticket holds -->

- [[01] Backend Runtime & Framework Evaluation](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/01-backend-runtime-evaluation.md): Reject PocketBase due to 32MB upload RAM buffering, SSD /tmp spillover, and WAL telemetry that defeats HDD spin-down; build a bespoke Go service using Chi v5 and `mattn/go-sqlite3` for an 8–14MB idle footprint and 0-RAM streaming direct to `/mnt/storage`.
- [[02] Low-Resource Media Processing Pipeline](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/02-low-resource-media-pipeline.md): Adopt isolated CLI subprocesses (`vipsthumbnail` + fast-seek `ffmpeg`) bound to Core 1 (`nice -n 19`, `ionice -c 3`, `prlimit 512MB`) with eager-on-upload processing to keep Go heap <25MB and protect HDD spin-down.
- [[03] Frontend Architecture & Single-Binary PWA Delivery](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/issues/03-frontend-pwa-architecture.md): Embed SvelteKit SPA into Go binary via `//go:embed all:build` with build-time precompression; sustain 60fps mobile gallery scrolling with $O(1)$ uniform-row Svelte 5 DOM virtualization (~40 DOM nodes, <25MB RAM); enforce 500-entry FIFO client Service Worker Preview Cache to insulate HDD spin-down without storage exhaustion; load PhotoSwipe v5 on-demand for touch gestures and inline video.

## Not yet specified

- **Job Queue & Background Worker Architecture**: SQLite-backed persistent queue for thumbnail generation, drive health checks, and scheduled backups with strict concurrency limits (single worker on 2-core CPU).
- **On-the-fly Streaming Zip Archives**: Zero-temp-file streaming zip generator for batch downloads directly from HDD to HTTP response body.
- **REST API Specification**: Contract definition between Go backend and SvelteKit frontend for galleries, albums, auth, and link sharing.
- **User Invitation & Account Recovery**: Low-overhead onboarding flow for non-technical community users without complex SMTP requirements.

## Out of scope

- Collaborative online document editing (OnlyOffice, Collabora, WOPI integrations).
- Heavy background desktop sync daemons with two-way file conflict resolution.
- AI/ML features (facial recognition, object classification, vector embedding search).
- Multi-node clustering or distributed consensus algorithms (Arkive is single-node/homelab-first).
