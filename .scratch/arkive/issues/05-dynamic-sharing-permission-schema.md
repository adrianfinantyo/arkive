# Dynamic Sharing & Permission Engine Schema

Type: grilling
Status: resolved
Blocked by: 01

## Question

How should the relational database schema and permission engine model hierarchical folder ownership, dynamic share links, and user access roles (Admin, Uploader, Viewer) without recursive database query penalties or WebSocket overhead?

Specific decisions to resolve with the user:
1. **Dynamic Link Semantics:** When an Uploader shares a folder via a public link, and subsequently uploads new photos to that folder, how should the share link query resolve those new items upon page reload? Does the share link point to a persistent folder ID or an immutable snapshot?
2. **Permission Inheritance & Path Traversal:** How are folder hierarchies represented in the database? (e.g., Materialized Path `parent_path`, Adjacency List `parent_id`, or Nested Sets)? Which representation ensures fast permission checks and zero-cost subfolder listing on low-power CPUs?
3. **Link Access Controls:** How should optional password protection (e.g. Argon2id vs bcrypt with tuned cost for Pentium E5400) and link expiration dates be validated and stored?
4. **Viewer Capabilities:** Can Viewers who access a shared folder download individual media items, full albums (via streaming zip), or reshuffle views (grid vs. chronological list)?

## Answer

**Verdict: Implement direct column visibility with Materialized Path traversal, three-tier role-based access with Group sharing, eager dual-format preview generation (WebP gallery + JPEG downloads) on SSD, and resource-governed video streaming.**

### Agreed Decisions
1. **Direct Column Visibility & Zero Share Tables**:
   - Sharing attributes (`visibility` ENUM('private', 'group', 'public'), unique `share_token`) reside directly on `folders` (and individual `media` items).
   - Dynamic resolution queries the target folder and its subtree without separate share-link join tables or per-file membership records.
   - Any items uploaded to a shared folder appear dynamically upon page reload.
2. **Materialized Path Traversal**:
   - Folder hierarchies use a `path` column (e.g. `/root_id/sub_id/`) with a B-tree index alongside `parent_id`.
   - Single prefix queries (`WHERE path LIKE '/<folder_id>/%'`) resolve entire subtrees and verify access without recursive CTEs on both PostgreSQL and SQLite.
3. **Role Boundaries & Group Capabilities**:
   - **Admin**: Full system configuration, visibility over all hosted assets, and sole authority to create and manage Groups. Admins can edit, move, or delete any asset.
   - **Uploader**: Owns folders and media files (`owner_id`), manages share links. When assigned to a Group, Uploaders can view and download all group assets. Only the owner or an Admin can edit, rename, move, or delete assets.
   - **Viewer**: Read-only access to assigned groups or public share links.
   - **Link Visibility**: Private (owner and group members only) or public (accessible by anyone with the link token). Passwords are omitted for simplicity.
4. **Dual-Format Previews & Universal Downloads**:
   - **Browser Gallery**: WebP previews served exclusively from the SSD Preview Cache (`/var/cache/arkive/previews/`) to maximize mobile rendering efficiency.
   - **Universal Downloads**:
     - `optimized` (default): Serves pre-generated compressed standard JPEG files (e.g. 1600px max dimension, standard sRGB) created eagerly at upload time into the SSD Preview Cache. Single and batch zip downloads produce standard `.jpg` files universally compatible with any mobile device or social platform, serving 100% from SSD without waking the HDD.
     - `original`: Serves full-resolution raw media from the Raw Media Store on `/mnt/storage`, displaying the "Preparing media..." indicator to absorb spin-up latency.
5. **Video Limits & Pre-Download Validation**:
   - Videos are never transcoded on the host; video downloads and streams always pull original files from `/mnt/storage`.
   - Resource limits (`MAX_VIDEO_ZIP_SIZE_MB` and concurrent stream quotas) are backed by dynamic checks on host memory and CPU load.
   - If a selected video exceeds the size limit for batch zip downloads, the UI alerts the user upfront to deselect the video and download it individually.

