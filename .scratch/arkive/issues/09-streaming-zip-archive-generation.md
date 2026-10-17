# On-the-Fly Streaming Zip Archive Generation

Type: grilling
Status: open
Blocked by: 05, 06

## Question

How should Arkive generate on-the-fly zip archives for batch and album downloads directly from disk to the HTTP response stream without writing temporary `.zip` files to SSD or buffering multi-gigabyte payloads in RAM?

Specific decisions to resolve with the user:
1. **Streaming Compression Mode:** Should zip streams use Store mode (zero compression, pure sequential I/O to maximize download speed and eliminate CPU usage on the Pentium E5400) or Deflate compression (`flate.BestSpeed` / level 1)?
2. **Dual-Tier Stream Routing:** How should the zip generator resolve file paths based on the `quality` parameter (pulling WebP/JPEG files from `/var/cache/arkive/previews/` for `optimized` vs raw files from `/mnt/storage/library/` for `original`)?
3. **Pre-Flight Validation & Error Handling:** How should file counts, total uncompressed size, and oversized video warnings be evaluated and returned before the HTTP stream starts?
4. **Client Cancellation & Connection Drops:** When a user cancels a download or the connection drops halfway through a 5GB archive, how does the Go runtime terminate the disk reader and reclaim resources immediately?
