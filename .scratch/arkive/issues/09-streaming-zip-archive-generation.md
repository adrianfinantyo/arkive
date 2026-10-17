# On-the-Fly Streaming Zip Archive Generation

Type: grilling
Status: resolved
Blocked by: 05, 06

## Question

How should Arkive generate on-the-fly zip archives for batch and album downloads directly from disk to the HTTP response stream without writing temporary `.zip` files to SSD or buffering multi-gigabyte payloads in RAM?

Specific decisions to resolve with the user:
1. **Streaming Compression Mode:** Should zip streams use Store mode (zero compression, pure sequential I/O to maximize download speed and eliminate CPU usage on the Pentium E5400) or Deflate compression (`flate.BestSpeed` / level 1)?
2. **Dual-Tier Stream Routing:** How should the zip generator resolve file paths based on the `quality` parameter (pulling WebP/JPEG files from `/var/cache/arkive/previews/` for `optimized` vs raw files from `/mnt/storage/library/` for `original`)?
3. **Pre-Flight Validation & Error Handling:** How should file counts, total uncompressed size, and oversized video warnings be evaluated and returned before the HTTP stream starts?
4. **Client Cancellation & Connection Drops:** When a user cancels a download or the connection drops halfway through a 5GB archive, how does the Go runtime terminate the disk reader and reclaim resources immediately?

## Answer

**Verdict: Stream on-the-fly zip archives directly to HTTP responses using Store mode (compression level 0) with 64KB pooled buffers and zip data descriptors. Route file reads based on download quality to serve SSD-cached JPEG previews without waking the mechanical drive, enforce pre-flight bounds (1,000 files / 10GB limit), and monitor request context cancellation to immediately abort disk reads and release semaphores.**

### Agreed Decisions
1. **Store Mode with Streaming Data Descriptors**:
   - Use Store mode (`zip.Store`, method 0, zero compression) exclusively.
   - Eliminates CPU overhead on Core 0, keeping CPU usage under 2% and sustaining wire speeds (100+ MB/s) on already compressed media (JPEG, WebP, MP4).
   - Set bit 3 (0x0008) in the local file header general purpose bit flag to write streaming data descriptors (CRC-32, sizes) immediately following each file payload.
   - Stream through pooled 64KB buffers (`sync.Pool`), maintaining stream heap memory under 100KB.
2. **Dual-Tier Path Routing & Spin-Down Preservation**:
   - `quality=optimized` (Default for Viewer role and public Share Links): Routes 100% of photo assets to pre-generated 1600px JPEG previews in the SSD Preview Cache (`/var/cache/arkive/previews/`). Secondary mechanical drive remains in low-power spin-down. Optimized batch downloads omit videos to avoid spin-up; videos require selecting original quality.
   - `quality=original`: Resolves paths from the Raw Media Store (`/mnt/storage/library/`), waking the secondary drive with a "Preparing media..." UI indicator and acquiring a slot from a 2-stream disk read semaphore.
   - Resolve all file paths from database metadata on SSD without filesystem `stat` calls on `/mnt/storage`.
3. **Pre-Flight Validation & Mid-Stream Error Handling**:
   - Execute all authorization and boundary validations before calling `w.WriteHeader(http.StatusOK)`.
   - Validate read permissions using Materialized Path queries, check file existence, and enforce caps: max 1,000 files, max 10GB total uncompressed size, max 1GB per individual video. Exceeding requests return HTTP 400 or HTTP 413 Problem Details.
   - On mid-stream disk read or network write failures, terminate the socket connection immediately without writing the Central Directory or End of Central Directory (EOCD) records, causing client extractors to cleanly detect corrupted transfers.
4. **Context Cancellation & Resource Reclamation**:
   - Bind every chunk read and HTTP flush to `r.Context().Done()`.
   - When the client disconnects or cancels, exit the loop immediately, close the active file descriptor (scoped per file to prevent descriptor leaks), recycle the 64KB buffer into `sync.Pool`, and release the disk read semaphore via `defer`.
