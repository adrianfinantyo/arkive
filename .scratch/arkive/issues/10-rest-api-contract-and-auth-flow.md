# REST API Specification & Contract Design

Type: grilling
Status: resolved
Blocked by: 05

## Question

How should Arkive structure its REST API contracts, JSON response envelopes, error handling, and SvelteKit API client bindings for galleries, albums, media items, auth, and link sharing?

Specific decisions to resolve with the user:
1. **Response Envelope & Error Format:** Should Arkive use a standardized JSON wrapper (`{ "data": ..., "error": null }`) or return flat JSON models with RFC 7807 Problem Details for errors?
2. **Infinite Scroll Pagination:** Should gallery listing use cursor-based pagination (keyed on `(taken_at, id)` for deterministic $O(1)$ scrolling through 50,000 photos) or traditional offset/limit pagination?
3. **HTTP Caching & ETag Validation:** How should HTTP `ETag` and `If-None-Match` headers be leveraged on album and gallery listings to return instant HTTP 304 Not Modified responses without repeated database querying?
4. **Type Sharing with Frontend:** Should Go domain structs generate TypeScript types automatically for SvelteKit using a code generator, or should API contracts be defined manually?

## Answer

**Verdict: Adopt flat JSON response models paired with RFC 7807 Problem Details for errors, enforce cursor-based pagination keyed on `(taken_at, id)` for constant-time gallery navigation, provide lightweight ETag validation using Valkey timeline revision keys to return HTTP 304 without database queries, and automate TypeScript type generation from Go domain models using `tygo` via `go generate`.**

### Agreed Decisions
1. **Flat Models, Collection Envelope & RFC 7807 Errors**:
   - Single resources return flat JSON models (`MediaItemDTO`, `AlbumDTO`).
   - List endpoints use a dedicated collection envelope: `{ items: T[], next_cursor: string | null, has_more: boolean }`.
   - All 4xx and 5xx errors use RFC 7807 Problem Details (`application/problem+json`) with standard fields (`type`, `title`, `status`, `detail`, `instance`) and an optional `errors` map for validation failures.
   - Eliminates generic wrapper boxing allocations in Go on Core 0 and avoids nested unboxing in SvelteKit TypeScript types.
2. **Keyset Cursor Pagination on `(taken_at, id)`**:
   - Paginate media galleries using a composite B-tree index on `(taken_at DESC, id DESC)`.
   - Cursor is an opaque URL-safe Base64 string encoding `taken_at_micros:id`.
   - Query descends index directly: `WHERE (taken_at, id) < ($1, $2) ORDER BY taken_at DESC, id DESC LIMIT 50`.
   - Delivers deterministic $O(1)$ query speed under 2ms for 50,000+ items and prevents photo duplication or skipped rows during active uploads.
3. **Valkey Atomic Version Counters & ETag Validation**:
   - Track atomic revision integers in Valkey: `user:{id}:timeline:version` and `folder:{id}:version`.
   - Mutations (uploads, edits, deletions) execute an atomic `INCR` on the corresponding key.
   - Endpoints emit weak ETags: `W/"tl-<version>-<limit>-<cursor_hash>"` with `Cache-Control: private, no-cache`.
   - Chi middleware evaluates `If-None-Match` in under 0.2ms against Valkey, returning HTTP 304 Not Modified without querying PostgreSQL/SQLite or serializing JSON. Static previews use immutable 1-year caching.
4. **Automated Type Generation via `tygo`**:
   - Target explicit API DTO structs in `internal/api/dto/` using `tygo` executed via `go generate ./...`.
   - Emits TypeScript definitions directly to `frontend/src/lib/types/api.gen.ts`.
   - Keeps internal GORM database models isolated from frontend API visibility without heavy external CLI dependencies.
