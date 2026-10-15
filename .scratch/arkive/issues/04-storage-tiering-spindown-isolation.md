# Storage Tiering & Drive Spin-Down Isolation

Type: grilling
Status: resolved
Blocked by: 01, 02

## Question

How should Arkive manage the boundaries between the SSD (`/`) and HDD (`/mnt/storage`) to guarantee that the mechanical hard drive remains spun down during routine browsing and gallery inspection?

Specific decisions to resolve with the user:
1. **Eager vs. Lazy Thumbnail Generation:** Should previews be generated eagerly at upload time (waking the HDD during upload, but guaranteeing that all subsequent gallery views only touch SSD), or lazily on first access (which would wake the HDD on first view if a thumbnail is missing)?
2. **Metadata Isolation:** What exact file metadata must be stored in the SQLite database on SSD (e.g., file sizes, dimensions, MIME types, EXIF dates, folder hierarchies) to completely eliminate directory traversal/stat syscalls on `/mnt/storage`?
3. **Spin-Up User Experience:** When a user requests a full-resolution original download or raw video playback from a spun-down HDD (which can take 5-15 seconds to spin up), how should the UI communicate this delay to prevent user frustration or timeout errors?
4. **Preview Cache Lifecycle & Eviction:** How should the SSD preview cache size be bounded (e.g., LRU eviction or quota-based pruning), and how does Arkive regenerate evicted thumbnails without thrashing the HDD?

## Answer

**Verdict: Isolate routine operations 100% to the SSD using an eager-on-upload thumbnail pipeline, complete metadata denormalization in SQLite, consumer-friendly spin-up feedback ("Preparing media..."), and synchronized thumbnail lifecycle management.**

### Agreed Decisions
1. **Eager Thumbnail Generation on Upload**: Media previews are enqueued immediately upon upload completion while the mechanical HDD is already spun up. Generated WebP thumbnails are committed directly to the SSD Preview Cache (`/var/cache/arkive/previews/`). All subsequent gallery, album, and timeline navigation by any number of viewers hits only SSD and SQLite, leaving the mechanical HDD in low-power spin-down indefinitely.
2. **Strict Metadata Isolation in SQLite**: Structural and media metadata (file sizes, image/video dimensions, MIME types, EXIF dates, folder hierarchies, and thumbnail hashes) are stored exclusively in SQLite (`arkive.db` on SSD). The web server is strictly forbidden from executing `os.Stat` or directory traversal syscalls on `/mnt/storage` during browsing, preventing accidental drive spin-ups.
3. **Consumer-Friendly Spin-Up UX**: When a user requests a raw download or uncompressed video stream from a spun-down drive, the UI displays a warm, non-technical feedback message: *"Preparing media..."* or *"Preparing video..."* with an extended client and server HTTP timeout (30–60s) to absorb mechanical spin-up latency smoothly.
4. **Synchronized Lifecycle & Idempotency**: Preview thumbnails are retained permanently on SSD by default (averaging only ~2.5GB for 50,000 photos). When a file is deleted (moved to trash/bin), its thumbnail remains accessible in the trash view; upon permanent deletion (either manual or via the $X$-day auto-purge), an idempotent cleanup routine deletes both the raw media file on `/mnt/storage` and its associated WebP preview files on the SSD.
