# Low-Resource Media Processing Pipeline

Type: research
Status: open
Blocked by: None

## Question

How should Arkive generate lightweight WebP image previews and extract video keyframes on an Intel Pentium Dual-Core E5400 (2 cores, 2 threads, 2.70GHz, no Intel QuickSync hardware acceleration) while keeping CPU usage bounded, preventing RAM thrashing, and avoiding disk I/O bottlenecks?

Specifically investigate:
1. **Tool Selection for Images:** Compare pure Go image libraries (e.g., `golang.org/x/image/webp`, `disintegration/imaging`), `libvips` bindings (e.g., `bimg` - requires CGO), and direct CLI execution (`cwebp` / `vips`) in terms of memory overhead and CPU speed on dual-core CPUs.
2. **Video Keyframe Extraction:** How to extract a single representative video frame (e.g., at 1s or 10% duration) using `ffmpeg` CLI with minimal CPU impact (`-ss` before input, `-frames:v 1`, nice/ionice process priority)?
3. **Queue & Resource Throttling:** How should thumbnail generation jobs be scheduled to ensure Arkive never consumes more than 1 core for processing, avoiding system unresponsiveness and keeping memory consumption strictly under control?
4. **Conclusion & Recommendation:** What is the optimal media pipeline architecture and command/library choice for this legacy hardware?
