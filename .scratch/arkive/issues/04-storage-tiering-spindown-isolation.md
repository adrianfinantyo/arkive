# Storage Tiering & Drive Spin-Down Isolation

Type: grilling
Status: open
Blocked by: 01, 02

## Question

How should Arkive manage the boundaries between the SSD (`/`) and HDD (`/mnt/storage`) to guarantee that the mechanical hard drive remains spun down during routine browsing and gallery inspection?

Specific decisions to resolve with the user:
1. **Eager vs. Lazy Thumbnail Generation:** Should previews be generated eagerly at upload time (waking the HDD during upload, but guaranteeing that all subsequent gallery views only touch SSD), or lazily on first access (which would wake the HDD on first view if a thumbnail is missing)?
2. **Metadata Isolation:** What exact file metadata must be stored in the SQLite database on SSD (e.g., file sizes, dimensions, MIME types, EXIF dates, folder hierarchies) to completely eliminate directory traversal/stat syscalls on `/mnt/storage`?
3. **Spin-Up User Experience:** When a user requests a full-resolution original download or raw video playback from a spun-down HDD (which can take 5-15 seconds to spin up), how should the UI communicate this delay to prevent user frustration or timeout errors?
4. **Preview Cache Lifecycle & Eviction:** How should the SSD preview cache size be bounded (e.g., LRU eviction or quota-based pruning), and how does Arkive regenerate evicted thumbnails without thrashing the HDD?
