# Streaming Upload & Buffer Control Architecture

Type: grilling
Status: resolved
Blocked by: 01

## Question

How should Arkive design its upload ingestion pipeline to stream multi-gigabyte media files directly to `/mnt/storage` without buffering file chunks in RAM or exhausting SSD space with temporary files?

Specific decisions to resolve with the user:
1. **Upload Protocol:** Should Arkive support resumable uploads via the open `tus` protocol (using tusd-go or a lightweight native Go implementation), standard HTTP Chunked Transfer Encoding, or a simple chunked multipart REST endpoint?
2. **Buffer Allocation & Memory Limits:** How should the I/O buffer be sized (e.g. 32KB to 64KB fixed buffers in `io.CopyBuffer`) to ensure concurrent uploads by multiple users never exceed the 100MB RAM ceiling?
3. **Partial & Interrupted Uploads:** Where are incomplete upload chunks held? Should they be written directly to a `.staging/` directory on `/mnt/storage` to prevent wasting SSD writes, and how are abandoned uploads cleaned up?
4. **Direct Handoff to Preview Pipeline:** Once an upload completes and the file is synced to disk, how does the system trigger the thumbnail generation task without blocking the HTTP upload response?

## Answer

**Verdict: Implement a lightweight `tus` (v1.0) resumable upload engine streaming directly to `/mnt/storage/.staging/` with pooled 64KB buffers, a 2-writer disk semaphore with round-robin chunk queueing, `fallocate()` contiguous block allocation, and read-prioritized 5-stream video governance.**

### Agreed Decisions
1. **Tus Resumable Protocol & Direct Streaming**:
   - Support open `tus` v1.0 protocol via a native Go HTTP handler using `PATCH` with `Upload-Offset`.
   - Stream request bodies straight to disk with a pooled 64KB buffer (`sync.Pool`), completely bypassing root SSD `/tmp` buffering and keeping upload heap memory under 1MB.
   - Completed files move to their target library directory via an atomic, zero-copy `os.Rename` on `/mnt/storage`. Abandoned staging files older than 24 hours are purged daily via an Asynq worker.
2. **HDD Write Concurrency & Thrashing Protection**:
   - Limit active simultaneous disk writes to **2 concurrent streams** using a Go semaphore.
   - Acquire the semaphore per chunk write window rather than per file, enabling fair round-robin progress across multiple concurrent uploaders without disk head thrashing.
   - Call `fallocate()` upon upload creation to pre-allocate contiguous disk blocks, preventing ext4 filesystem fragmentation.
   - Check free disk space before session creation: reject uploads (HTTP 507 Insufficient Storage) if free space falls below a 5GB safety margin.
   - Isolate each upload in a UUID-named file protected by an in-memory per-upload mutex.
3. **Video Stream Prioritization & Resource Limits**:
   - Prioritize video playback reads over background upload writes using the Linux `mq-deadline` I/O scheduler.
   - Serve HTTP 206 range requests in 2MB read-ahead chunks so clients buffer 20 to 30 seconds of playback upfront.
   - Cap active raw video streams to a maximum of 5 concurrent viewers, displaying "Preparing video..." during peak I/O saturation.
4. **Asynchronous Preview Pipeline Handoff**:
   - Return an HTTP 201/200 success response to the client immediately upon file commit and database metadata insertion.
   - Eagerly enqueue thumbnail tasks to Valkey/Asynq so `arkive-worker` generates WebP and JPEG previews while the mechanical drive is already spun up from the upload write.

