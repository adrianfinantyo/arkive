# Dynamic Sharing & Permission Engine Schema

Type: grilling
Status: claimed
Blocked by: 01

## Question

How should the relational SQLite schema and permission engine model hierarchical folder ownership, dynamic share links, and user access roles (Admin, Uploader, Viewer) without recursive database query penalties or WebSocket overhead?

Specific decisions to resolve with the user:
1. **Dynamic Link Semantics:** When an Uploader shares a folder via a public link, and subsequently uploads new photos to that folder, how should the share link query resolve those new items upon page reload? Does the share link point to a persistent folder ID or an immutable snapshot?
2. **Permission Inheritance & Path Traversal:** How are folder hierarchies represented in SQLite? (e.g., Materialized Path `parent_path`, Adjacency List `parent_id`, or Nested Sets)? Which representation ensures fast permission checks and zero-cost subfolder listing on low-power CPUs?
3. **Link Access Controls:** How should optional password protection (e.g. Argon2id vs bcrypt with tuned cost for Pentium E5400) and link expiration dates be validated and stored?
4. **Viewer Capabilities:** Can Viewers who access a shared folder download individual media items, full albums (via streaming zip), or reshuffle views (grid vs. chronological list)?
