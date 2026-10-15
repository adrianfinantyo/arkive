# Backend Runtime & Framework Evaluation

Type: research
Status: resolved
Blocked by: None

## Question

Evaluate Go backend runtime options for Arkive: PocketBase (as a framework/library) vs. a bespoke Go service (using standard library `net/http` or lightweight routing like Chi, with `modernc.org/sqlite` or `mattn/go-sqlite3`).

Specifically investigate:
1. **Memory & Idle Footprint:** Does PocketBase run reliably under 20-40MB idle RAM, leaving ample headroom below the strict 100MB system threshold on Linux x86-64, or does it carry framework overhead that risks exceeding the limit under load?
2. **Streaming I/O Control:** Does PocketBase allow intercepting multipart/stream uploads to pipe directly to an open disk file descriptor on `/mnt/storage` without buffering file bodies in memory or `/tmp` on SSD? How easily does a bespoke Go HTTP handler achieve zero-RAM streaming?
3. **Binary Footprint & Maintainability:** Compare the developer velocity of PocketBase's built-in auth, admin dashboard, and migration engine against the minimalism and fine-grained hardware control of a bespoke Go service.
4. **Conclusion & Recommendation:** Which backend architecture best aligns with the low-resource legacy hardware constraints (Intel Pentium E5400, 3.8GB RAM)?

## Answer

**Verdict: Reject PocketBase; build a bespoke Go service using `go-chi/chi/v5` and `mattn/go-sqlite3`.**

Full research report: [`.scratch/arkive/research/01-backend-runtime-evaluation.md`](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/research/01-backend-runtime-evaluation.md).

### Key Findings
1. **Memory Budget (<100MB)**: PocketBase idles at 22–36MB and climbs to 55–85MB under concurrent gallery browsing (due to Goja JSVM, dynamic JSON schema reflection, SSE broker, and dual SQLite connection pools), leaving <15MB margin before Linux `oom-killer`. A bespoke Chi service idles at 8–14MB and stays under 28MB under load, guaranteeing >70MB headroom.
2. **Streaming Uploads & SSD Wear**: PocketBase uses Go's `ParseMultipartForm(32 << 20)`, buffering 32MB in RAM per upload and spilling excess bytes to `/tmp` on the root SSD (`/`). A 4GB video upload writes 4GB to SSD, reads it back, and writes to HDD (8–12GB total write amplification). A bespoke Go handler uses `r.MultipartReader()` with `io.CopyBuffer` using a fixed 64KB buffer directly to `/mnt/storage/.staging/`: exactly 64KB RAM, 0 bytes written to SSD, and 1x sequential write to HDD.
3. **SQLite Engine on Pentium E5400**: `modernc.org/sqlite` is machine-translated C-to-Go, causing heavy GC pressure and 1.5x–3.0x execution slowdowns on a 2-core Wolfdale CPU with 2MB L2 cache. `mattn/go-sqlite3` uses native C SQLite via GCC; page cache lives in C heap via `malloc` (completely invisible to Go GC), delivering deterministic microsecond queries.
4. **Drive Spin-Down Protection**: PocketBase flushes request logs to `auxiliary.db` in WAL mode every few seconds, which prevents the HDD from spinning down if data lives on the storage drive. A bespoke service isolates `arkive.db` and WebP Preview Cache entirely on SSD (`/`), touching `/mnt/storage` only for uploads or original file downloads.
