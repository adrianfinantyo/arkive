# Frontend Architecture & Single-Binary PWA Delivery

Type: research
Status: resolved
Blocked by: None

## Question

How should Arkive's frontend and PWA be structured to deliver an instant, responsive mobile experience while running as a single embedded binary and conserving client/server resources?

Specifically investigate:
1. **Packaging & Single-Binary Distribution:** How to build SvelteKit with `@sveltejs/adapter-static` (SPA mode) and bundle the output directly into the Go backend binary using Go 1.16+ `embed.FS`, eliminating any Node.js runtime on the server.
2. **Mobile Gallery Performance & Virtualization:** How can a mobile web client smoothly scroll through galleries with thousands of images/videos without excessive DOM node overhead or memory consumption? Evaluate lightweight virtual scrolling approaches (e.g., svelte-virtual or simple intersection-observer based windowing).
3. **PWA & Offline Asset Strategy:** What caching strategy should the service worker adopt to cache app shell, icons, and recent thumbnails on client devices without exhausting low-end mobile phone storage?
4. **Touch Gestures & Media Viewer:** What minimal library or vanilla approach should power the lightbox viewer (swipe to navigate, pinch to zoom, inline HTML5 video playback)?

## Answer

**Verdict: Deliver an embedded SvelteKit SPA via Go `//go:embed all:build` with build-time precompression, an $O(1)$ uniform-row Svelte 5 virtual grid, a 500-entry FIFO Service Worker Preview Cache, and an on-demand dynamically imported PhotoSwipe v5 lightbox.**

Full research report: [`.scratch/arkive/research/03-frontend-pwa-architecture.md`](file:///home/adrian/adrian-dev/arkive/.scratch/arkive/research/03-frontend-pwa-architecture.md).

### Key Findings
1. **Single-Binary Packaging & SPA Fallback**: SvelteKit is built via `@sveltejs/adapter-static` with `fallback: 'index.html'`, `ssr = false`, and `precompress: true`. The output is compiled into the Go backend binary using `//go:embed all:build` (the `all:` prefix is mandatory to embed `_app/` which standard wildcard ignores). A Go `http.Handler` serves immutable hashed assets with 1-year cache headers, matches `Accept-Encoding` to precompressed `.br` / `.gz` files (zero runtime CPU overhead on Pentium E5400), and routes non-file paths to `index.html` (HTTP 200) for client-side routing.
2. **Mobile Gallery Performance & Texture Memory**: Offscreen CSS `content-visibility: auto` skips layout/paint but retains decoded bitmap textures ($W \times H \times 4$ bytes) in RAM, crashing mobile browser tabs (1,000 unvirtualized thumbnails = 640MB RAM). A bespoke Svelte 5 uniform-row virtual grid performs $O(1)$ windowing math, mounting only visible rows + 3 overscan rows (~36–48 `<img>` DOM nodes, <25MB RAM) with `loading="lazy"` and `decoding="async"`, sustaining 60fps scrolling on budget phones.
3. **PWA Service Worker & Quota Safety**: Using SvelteKit's native `src/service-worker.ts`, the PWA maintains two distinct caches: a deployment-pinned App Shell cache (precached on install, purged on activate) and a dedicated 500-entry / 50MB **Preview Cache** for WebP thumbnails. Thumbnail eviction uses zero-dependency FIFO pruning based on W3C `Cache.keys()` insertion order. Client thumbnail caching eliminates server network requests during repeated browsing, protecting backend **Drive Spin-Down**. Raw media streams (`/mnt/storage`) bypass Cache Storage entirely.
4. **Touch Gestures & Video Decoder Cleanup**: PhotoSwipe v5 (~16.6 kB gzipped, 0 dependencies) is loaded on-demand via dynamic `import()` only when a thumbnail is clicked (costing 0 KB on initial gallery load). It provides hardware-accelerated pinch-to-zoom, pan, and swipe-to-dismiss. Videos play inline via `playsinline` and `muted`. On slide deactivation, `video.removeAttribute('src')` and `video.load()` are invoked to immediately free the mobile SoC's limited hardware video decoder channels.
