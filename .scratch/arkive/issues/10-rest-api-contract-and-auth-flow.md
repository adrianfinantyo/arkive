# REST API Specification & Contract Design

Type: grilling
Status: open
Blocked by: 05

## Question

How should Arkive structure its REST API contracts, JSON response envelopes, error handling, and SvelteKit API client bindings for galleries, albums, media items, auth, and link sharing?

Specific decisions to resolve with the user:
1. **Response Envelope & Error Format:** Should Arkive use a standardized JSON wrapper (`{ "data": ..., "error": null }`) or return flat JSON models with RFC 7807 Problem Details for errors?
2. **Infinite Scroll Pagination:** Should gallery listing use cursor-based pagination (keyed on `(taken_at, id)` for deterministic $O(1)$ scrolling through 50,000 photos) or traditional offset/limit pagination?
3. **HTTP Caching & ETag Validation:** How should HTTP `ETag` and `If-None-Match` headers be leveraged on album and gallery listings to return instant HTTP 304 Not Modified responses without repeated database querying?
4. **Type Sharing with Frontend:** Should Go domain structs generate TypeScript types automatically for SvelteKit using a code generator, or should API contracts be defined manually?
