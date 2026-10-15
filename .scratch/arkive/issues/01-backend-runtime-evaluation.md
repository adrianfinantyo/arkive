# Backend Runtime & Framework Evaluation

Type: research
Status: open
Blocked by: None

## Question

Evaluate Go backend runtime options for Arkive: PocketBase (as a framework/library) vs. a bespoke Go service (using standard library `net/http` or lightweight routing like Chi, with `modernc.org/sqlite` or `mattn/go-sqlite3`).

Specifically investigate:
1. **Memory & Idle Footprint:** Does PocketBase run reliably under 20-40MB idle RAM, leaving ample headroom below the strict 100MB system threshold on Linux x86-64, or does it carry framework overhead that risks exceeding the limit under load?
2. **Streaming I/O Control:** Does PocketBase allow intercepting multipart/stream uploads to pipe directly to an open disk file descriptor on `/mnt/storage` without buffering file bodies in memory or `/tmp` on SSD? How easily does a bespoke Go HTTP handler achieve zero-RAM streaming?
3. **Binary Footprint & Maintainability:** Compare the developer velocity of PocketBase's built-in auth, admin dashboard, and migration engine against the minimalism and fine-grained hardware control of a bespoke Go service.
4. **Conclusion & Recommendation:** Which backend architecture best aligns with the low-resource legacy hardware constraints (Intel Pentium E5400, 3.8GB RAM)?
