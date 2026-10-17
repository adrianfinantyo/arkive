# Client Offline Synchronization & Action Replay

Type: grilling
Status: open
Blocked by: 03, 10

## Question

How should Arkive manage client-side state, queue user actions (such as favoriting, tagging, moving items, or creating albums) in IndexedDB during network loss, and reconcile mutations upon reconnecting to the Go backend?

Specific decisions to resolve with the user:
1. **IndexedDB Mutation Queue Schema:** How should offline actions be structured, prioritized, and persisted locally in the browser to survive browser restarts and tab closures?
2. **Replay & Conflict Reconciliation Strategy:** How should the client replay queued mutations upon network recovery? How are conflicts handled if a file was moved or deleted on the server while the client was offline?
3. **Optimistic UI Updates in Svelte 5:** How should local Svelte 5 runes reflect queued actions immediately in the UI without awaiting network roundtrips, and how are optimistic changes rolled back on failure?
4. **Offline Asset Scope & Storage Limits:** Beyond the 500-entry FIFO Service Worker Preview Cache (ADR-0003), how should offline media availability be bounded to prevent exhausting mobile browser storage?
