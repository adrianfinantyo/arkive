# 📦 Arkive

> **Ultra-lightweight, self-hosted media archive and file sharing web platform designed for legacy hardware, homelabs, and drive longevity.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Go Version](https://img.shields.io/badge/Go-1.22+-00ADD8?logo=go)](https://go.dev/)
[![SvelteKit](https://img.shields.io/badge/SvelteKit-2.0-FF3E00?logo=svelte)](https://kit.svelte.dev/)
[![Docker Compose](https://img.shields.io/badge/Docker_Compose-Ready-2496ED?logo=docker)](https://www.docker.com/)

---

## 🌟 Overview

**Arkive** is an open-source, resource-conscious media management and file sharing platform inspired by the clean, intuitive user experiences of Google Drive and Google Photos. 

Unlike modern enterprise self-hosted platforms that demand gigabytes of idle RAM and heavy background indexing daemons, Arkive is built from the ground up to **run smoothly on vintage, low-spec hardware** (such as an Intel Pentium Dual-Core with <4GB RAM) while delivering an instant, 60fps mobile gallery experience.

### 🚫 Explicit Functional Exclusions (No Enterprise Bloat)
- **NO** collaborative office suites, spreadsheets, or document editors (OnlyOffice / Collabora / WOPI).
- **NO** continuous two-way desktop file synchronization daemons.
- **NO** heavy machine-learning workloads (AI face recognition, object detection, or vector indexing).

---

## ⚡ Core Highlights & Architectural Principles

### 1. 💤 Two-Tier Storage & Mechanical Drive Spin-Down
Arkive is engineered specifically for homelabs pairing an **SSD** (system & cache) with a large **mechanical HDD** (bulk storage):
- **SSD (`/`)**: Hosts the application, database metadata, Valkey session cache, and generated WebP preview thumbnails.
- **HDD (`/mnt/storage`)**: Houses the primary **Raw Media Store** and upload staging.
- **Drive Spin-Down Isolation**: Browsing galleries, inspecting albums, and viewing thumbnails hits the SSD exclusively. The mechanical storage drive stays asleep in low-power standby mode, waking up only when an upload is staged or an original raw download is requested.
- **Consumer-Friendly Feedback**: When an original file is downloaded from a sleeping drive, the UI displays a warm *"Preparing media..."* indicator with extended timeouts to seamlessly absorb spin-up latency.

### 2. 🌊 Zero-RAM Direct-to-Disk Streaming
- Uploads bypass system RAM and SSD temp files entirely.
- Using `r.MultipartReader()` with fixed 64KB I/O buffers (`io.CopyBuffer`), multi-gigabyte video uploads stream directly to `/mnt/storage/.staging/`.
- Eliminates SSD write wear, prevents OOM kernel kills, and commits files via atomic $O(1)$ filesystem renames on completion.

### 3. 🧩 Togglable Two-Container Monorepo
Arkive maintains a single clean monorepo deployed as two distinct containers:
- **`arkive-server`**: High-performance Go web server (`Chi` router), embedded SvelteKit PWA (`//go:embed all:build`), authentication, permissions, and direct streaming uploads. Lightweight Alpine image (~25MB) with zero media CLI dependencies.
- **`arkive-worker`**: Optional background worker container running `hibiken/asynq`, `vipsthumbnail` (libjpeg-turbo DCT shrink-on-load), and `ffmpeg` (instant keyframe seeking). Capped cleanly via Docker Compose (`cpus: 1`, `mem_limit: 384m`) and togglable on demand via Compose profiles (`profiles: ["worker"]`).

### 4. 🗄️ Persistence Portability (Open-Closed Principle)
- Domain entities depend strictly on repository interfaces (`internal/domain/`).
- Persistence is implemented via **GORM** (`gorm.io/gorm`), allowing Arkive to switch seamlessly between **PostgreSQL**, **SQLite**, and **MySQL** without modifying core business rules.
- Fast session and token caching backed by **Valkey 8** (`valkey-go`).

### 5. 📱 High-Performance Mobile PWA
- **Uniform-Row DOM Virtualization**: Mathematical $O(1)$ windowing in Svelte renders only ~40–60 DOM nodes (<25MB client RAM) regardless of whether an album holds 50 or 50,000 photos, sustaining 60fps scrolling on budget phones.
- **Budget-Capped Service Worker Cache**: Dual-tier PWA service worker with a 500-entry FIFO thumbnail cache that protects the server's drive spin-down while preventing mobile device storage exhaustion.
- **On-Demand Lightbox**: PhotoSwipe v5 loaded dynamically on click with hardware-accelerated touch gestures (pinch-zoom, swipe-to-dismiss) and explicit video decoder cleanup.

### 6. 🔗 Dynamic Sharing & Role Model
- **3 Roles**:
  - **Admin**: Master administrative control, configuration, user management, and global archive visibility.
  - **Uploader (Editor)**: Uploads, manages, and owns files/folders; controls sharing permissions.
  - **Viewer**: Read-only consumer accessing granted albums, playing inline media, or viewing public share links.
- **Dynamic Shared Links**: Public and restricted folder links dynamically resolve active contents on page refresh/request without WebSocket overhead.
- **Granular Controls**: Optional password protection (cached in Valkey), expiration dates, download permissions, and customizable grid/list view presets.

### 7. 🗑️ Trash Bin Lifecycle ($X$-Day Retention)
- Accidental deletions move items to a Trash Bin (`deleted_at` timestamp).
- Users can restore items or click "Empty Trash".
- An automated daily Asynq cron worker permanently purges items older than $X$ days (default 30 days), cascading deletion atomically across the database, the raw file on `/mnt/storage`, and the WebP thumbnails on SSD.

---

## 🏗️ Architecture & Technology Stack

```
                          ┌────────────────────────────────────────────────────────┐
                          │                 DOCKER COMPOSE STACK                   │
                          │                                                        │
Browser / Mobile PWA ────►│  [arkive-server] (Web & API)                           │
                          │  - Go 1.22+ (Chi v5 router)                            │
                          │  - Embedded SvelteKit SPA (go:embed all:build)         │
                          │  - GORM Repository (Open-Closed Principle)             │
                          │  - Zero-RAM Direct Streaming Uploads                   │
                          │                                                        │
                          │  [arkive-worker] (Togglable Background Processor)      │
                          │  - hibiken/asynq task worker                           │
                          │  - vipsthumbnail (isolated image preview transcoding)  │
                          │  - ffmpeg CLI (instant single-frame video extraction)  │
                          │  - Docker bounds: cpus: '1.0', mem_limit: 384m         │
                          │                                                        │
                          │  [postgres:16-alpine]  (Metadata volume on SSD)        │
                          │  [valkey:8-alpine]     (Cache/Queue volume on SSD)     │
                          └────────────────────────────────────────────────────────┘
                                      │                             │
                     (Preview Cache & DB Writes)            (Raw Media Ingestion)
                                      ▼                             ▼
                              ┌───────────────┐             ┌───────────────┐
                              │   SSD (`/`)   │             │ HDD (/mnt/..) │
                              │ (Always Fast) │             │  (Spins Down) │
                              └───────────────┘             └───────────────┘
```

| Layer | Technology | Rationale |
| :--- | :--- | :--- |
| **Backend API** | Go 1.22+ & `go-chi/chi/v5` | Minimal memory footprint (~15MB RSS), zero allocation routing, standard library compatibility. |
| **Frontend** | SvelteKit + Tailwind CSS | Static SPA export (`@sveltejs/adapter-static`), precompressed (`.br`/`.gz`), embedded via `embed.FS`. |
| **Database** | PostgreSQL 16 (or SQLite) via GORM | Relational integrity, soft-delete hooks, and dialect portability across database engines. |
| **Cache & Queue** | Valkey 8 & `hibiken/asynq` | Official open-source Linux Foundation Redis fork; reliable background queue with cron purges. |
| **Image Processing** | `vipsthumbnail` (libvips) | Subprocess execution, zero Go heap allocation, libjpeg-turbo DCT shrink-on-load ($8\times$ CPU reduction). |
| **Video Extraction** | `ffmpeg` CLI | Fast input seek (`-ss` before `-i`), single keyframe extraction (`-frames:v 1`), isolated from API server. |
| **Media Lightbox** | PhotoSwipe v5 | Dynamically imported on item tap (~16KB gzipped), 60fps pinch-to-zoom, inline video decoder cleanup. |

---

## 📁 Repository Structure

Arkive is organized as a clean Go monorepo:

```
arkive/
├── cmd/
│   ├── server/               # Main entrypoint for arkive-server (API & Web)
│   └── worker/               # Main entrypoint for arkive-worker (Media & Cron tasks)
├── internal/
│   ├── api/                  # Chi router, HTTP handlers, middleware, JSON payloads
│   ├── domain/               # Core business models & repository interfaces (OCP)
│   ├── store/                # GORM repository implementation
│   ├── cache/                # Valkey client wrapper for session and token caching
│   ├── media/                # vipsthumbnail and ffmpeg subprocess executors
│   ├── worker/               # Asynq task definitions and handler registration
│   └── web/                  # Embedded SvelteKit static asset handler (embed.FS)
├── web/                      # SvelteKit PWA source code
│   ├── src/
│   │   ├── lib/              # UI components (virtual grid, lightbox, navigation)
│   │   └── routes/           # Client-side routes (+page.svelte)
│   └── package.json
├── deploy/
│   ├── docker-compose.yml    # Standard multi-container production deployment
│   ├── Dockerfile.server     # Lean static binary container (~25MB)
│   └── Dockerfile.worker     # Media processing container with ffmpeg and libvips
├── docs/
│   ├── adr/                  # Architectural Decision Records (ADRs 0001–0006)
│   └── agents/               # Development guidelines and issue tracking conventions
├── GLOSSARY.md               # Strict domain vocabulary
└── README.md
```

---

## 🚀 Quick Start (Docker Compose)

### 1. Clone the Repository
```bash
git clone https://github.com/adrianfinantyo/arkive.git
cd arkive
```

### 2. Configure Environment Variables
Copy the sample environment file:
```bash
cp deploy/.env.example .env
```
Key configuration settings:
```ini
# Storage Topology
SSD_CACHE_PATH=/var/cache/arkive/previews
HDD_STORAGE_PATH=/mnt/storage/raw
HDD_STAGING_PATH=/mnt/storage/.staging

# Database & Cache
DATABASE_URL=postgres://arkive:secret@postgres:5432/arkive?sslmode=disable
VALKEY_URL=valkey:6379

# Data Lifecycle
TRASH_RETENTION_DAYS=30
```

### 3. Launch with Docker Compose
```bash
# Start all core services (Server, Worker, PostgreSQL, Valkey)
docker compose -f deploy/docker-compose.yml up -d

# Or start in light mode without the background media worker:
docker compose -f deploy/docker-compose.yml --profile server-only up -d
```

Visit `http://localhost:8080` in your browser.

---

## 📜 Architectural Decisions (ADRs)

Key architectural decisions are documented under [`docs/adr/`](file:///home/adrian/adrian-dev/arkive/docs/adr/):
- **[ADR-0001](file:///home/adrian/adrian-dev/arkive/docs/adr/0001-bespoke-go-runtime-over-pocketbase.md)**: Bespoke Go Service with Chi over PocketBase.
- **[ADR-0002](file:///home/adrian/adrian-dev/arkive/docs/adr/0002-cli-subprocess-media-processing.md)**: CLI Subprocess Media Pipeline via `vipsthumbnail` & `ffmpeg`.
- **[ADR-0003](file:///home/adrian/adrian-dev/arkive/docs/adr/0003-embedded-sveltekit-spa-with-virtualized-gallery.md)**: Embedded SvelteKit SPA with Uniform-Row Virtual Grid.
- **[ADR-0004](file:///home/adrian/adrian-dev/arkive/docs/adr/0004-docker-compose-with-postgres-and-redis.md)**: Containerized Deployment with PostgreSQL and Valkey.
- **[ADR-0005](file:///home/adrian/adrian-dev/arkive/docs/adr/0005-orm-for-multi-database-portability.md)**: ORM Persistence Abstraction for Multi-Database Portability.
- **[ADR-0006](file:///home/adrian/adrian-dev/arkive/docs/adr/0006-monorepo-with-server-worker-split.md)**: Monorepo with Separate Server & Worker Deployments.

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
