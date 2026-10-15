# Arkive

Arkive is a self-hosted media archive and file sharing web application built for small teams, community groups, and homelab users who want to store photos/files and share event galleries with non-technical mobile users without melting old hardware.

Think of it as the simple, clean parts of Google Photos and Google Drive—galleries, albums, shareable links, and direct downloads—without the enterprise weight.

## Why Arkive?

Most self-hosted photo platforms are great, but they assume you are running a modern multi-core server with plenty of RAM. They bundle AI facial recognition, object detection, vector search, continuous file syncing daemons, and background indexing jobs that keep old CPUs pegged at 100% and mechanical hard drives spinning 24/7.

If your homelab server is an old desktop or an energy-saving box, those stacks are simply too heavy.

Arkive was born out of a specific, real-world setup:

- **CPU:** Intel Pentium Dual-Core E5400 @ 2.70GHz (2 cores, 2 threads)
- **RAM:** 3.8 GiB total (~3.2 GiB available)
- **Fast Storage (SSD):** 256GB SSD (mounted at `/`)
- **Bulk Storage (HDD):** 1TB Western Digital HDD (mounted at `/mnt/storage`)
- **OS:** Linux

The goal is simple: run comfortably on hardware like this, keep idle memory low, give non-technical friends and family an instant mobile gallery experience, and keep mechanical drives asleep when nobody is downloading raw files.

## What it does

- **Mobile-friendly gallery:** Fast photo browsing and inline video playback that works smoothly on low-end phones without draining memory.
- **Drive spin-down friendly:** Thumbnails and database queries hit the SSD. The mechanical storage drive stays in low-power sleep mode during routine browsing, only waking up when an upload comes in or an original file is downloaded.
- **Zero-RAM streaming uploads:** File uploads stream directly to disk on the storage drive in fixed 64KB chunks instead of buffering in system memory or thrashing the SSD with temporary files.
- **Dynamic share links:** Share a folder link with password protection or expiration. When you add new photos to the folder, viewers see them on refresh without websocket overhead.
- **Simple roles:** 
  - **Admin:** full system management and configuration.
  - **Uploader:** uploads and organizes own files, creates and manages shares.
  - **Viewer:** read-only view of shared galleries and direct downloads.
- **Trash bin with auto-purge:** No file versioning bloat. Deleted files sit in the trash for a set number of days (default 30) before permanent removal, with full restore support.

## What it deliberately does NOT do

To keep the system lightweight and reliable on legacy hardware, Arkive intentionally excludes:
- No document/spreadsheet editing (no OnlyOffice, Collabora, or office suites).
- No continuous two-way desktop file synchronization daemons.
- No AI/machine learning (no face grouping, object tagging, or heavy background indexing).

## Tech Stack

Arkive is organized as a monorepo split into two deployable services:

- **Server (`arkive-server`):** Go backend using `chi` router and an embedded SvelteKit web frontend (`go:embed`). Handles API requests, auth, permissions, and direct streaming uploads.
- **Worker (`arkive-worker`):** Background processor running `vipsthumbnail` for image thumbnails and `ffmpeg` for video keyframes. Can be throttled or toggled on/off independently.
- **Data & Cache:** PostgreSQL (persisted via GORM for multi-database portability) and Valkey (for session caching and worker task queue).

Both services deploy via Docker Compose with container volumes placed so database writes hit the SSD and never wake the bulk storage HDD.

## Status

Arkive is currently in active architecture design and specification. Key architectural decisions and benchmarks are documented under `docs/adr/`.
