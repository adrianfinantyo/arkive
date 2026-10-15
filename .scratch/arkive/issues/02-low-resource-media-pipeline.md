# Low-Resource Media Processing Pipeline

Type: research
Status: resolved
Blocked by: None

## Question

How should Arkive generate lightweight WebP image previews and extract video keyframes on an Intel Pentium Dual-Core E5400 (2 cores, 2 threads, 2.70GHz, no Intel QuickSync hardware acceleration) while keeping CPU usage bounded, preventing RAM thrashing, and avoiding disk I/O bottlenecks?

Specifically investigate:
1. **Tool Selection for Images:** Compare pure Go image libraries (e.g., `golang.org/x/image/webp`, `disintegration/imaging`), `libvips` bindings (e.g., `bimg` - requires CGO), and direct CLI execution (`cwebp` / `vips`) in terms of memory overhead and CPU speed on dual-core CPUs.
2. **Video Keyframe Extraction:** How to extract a single representative video frame (e.g., at 1s or 10% duration) using `ffmpeg` CLI with minimal CPU impact (`-ss` before input, `-frames:v 1`, nice/ionice process priority)?
3. **Queue & Resource Throttling:** How should thumbnail generation jobs be scheduled to ensure Arkive never consumes more than 1 core for processing, avoiding system unresponsiveness and keeping memory consumption strictly under control?
4. **Conclusion & Recommendation:** What is the optimal media pipeline architecture and command/library choice for this legacy hardware?

## Answer

**Verdict: Use isolated CLI subprocesses (`vipsthumbnail` for images, `ffmpeg` for video keyframes) with strict OS resource confinement (pinned to Core 1, `nice -n 19`, `ionice -c 3`) and an eager-on-upload processing queue.**

Full research report: [`.scratch/arkive/research/02-low-resource-media-pipeline.md`](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/research/02-low-resource-media-pipeline.md).

### Key Findings
1. **Image Tool Selection**: Pure Go (`golang.org/x/image/webp`) lacks a WebP encoder and inflates uncompressed 32-bit bitmaps in the Go GC heap (triggering 200–300MB heap spikes that break the <100MB limit). Cgo bindings (`bimg`) risk segfaulting the entire Go daemon on corrupted images. Subprocess execution via `vipsthumbnail` provides 100% process isolation, zero Go heap allocation, automatic libjpeg-turbo DCT shrink-on-load ($8\times$ reduction in CPU/RAM before resampling), and instant kernel page reclamation on process exit.
2. **Video Keyframe Extraction**: Fast input seek (`-ss 00:00:01` before `-i`) reads container indexes directly and seeks in <150ms (<500KB I/O) without decoding intermediate video. Combined with `-frames:v 1 -an -sn -dn`, proportional scaling (`scale=400:-2`), and single-pass WebP encoding (`-c:v libwebp -lossless 0 -q:v 75 -compression_level 3`), memory spikes are completely eliminated.
3. **CPU & I/O Isolation on Pentium E5400**:
   - Single worker goroutine pulling from an SQLite persistent queue (`media_jobs`).
   - Pinned strictly to CPU Core 1 via `taskset -c 1`, leaving Core 0 100% free for the Go HTTP server and SQLite queries.
   - Throttled with `nice -n 19` (minimum CPU weight) and `ionice -c 3` (idle I/O class), ensuring thumbnail generation never blocks UI requests or streaming uploads.
   - Guarded with `prlimit --as=512M --cpu=30` against decompression bombs.
4. **Drive Spin-Down Protection**: Thumbnails are generated eagerly immediately after upload while the secondary HDD is already active. Generated WebP previews are stored directly in the SSD Preview Cache (`/var/cache/arkive/previews/`). Subsequent mobile gallery browsing touches only SSD, preserving HDD spin-down.
