# Streaming Upload & Buffer Control Architecture

Type: grilling
Status: open
Blocked by: 01

## Question

How should Arkive design its upload ingestion pipeline to stream multi-gigabyte media files directly to `/mnt/storage` without buffering file chunks in RAM or exhausting SSD space with temporary files?

Specific decisions to resolve with the user:
1. **Upload Protocol:** Should Arkive support resumable uploads via the open `tus` protocol (using tusd-go or a lightweight native Go implementation), standard HTTP Chunked Transfer Encoding, or a simple chunked multipart REST endpoint?
2. **Buffer Allocation & Memory Limits:** How should the I/O buffer be sized (e.g. 32KB to 64KB fixed buffers in `io.CopyBuffer`) to ensure concurrent uploads by multiple users never exceed the 100MB RAM ceiling?
3. **Partial & Interrupted Uploads:** Where are incomplete upload chunks held? Should they be written directly to a `.staging/` directory on `/mnt/storage` to prevent wasting SSD writes, and how are abandoned uploads cleaned up?
4. **Direct Handoff to Preview Pipeline:** Once an upload completes and the file is synced to disk, how does the system trigger the thumbnail generation task without blocking the HTTP upload response?
