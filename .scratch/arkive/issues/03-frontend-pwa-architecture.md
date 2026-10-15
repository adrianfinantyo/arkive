# Frontend Architecture & Single-Binary PWA Delivery

Type: research
Status: open
Blocked by: None

## Question

How should Arkive's frontend and PWA be structured to deliver an instant, responsive mobile experience while running as a single embedded binary and conserving client/server resources?

Specifically investigate:
1. **Packaging & Single-Binary Distribution:** How to build SvelteKit with `@sveltejs/adapter-static` (SPA mode) and bundle the output directly into the Go backend binary using Go 1.16+ `embed.FS`, eliminating any Node.js runtime on the server.
2. **Mobile Gallery Performance & Virtualization:** How can a mobile web client smoothly scroll through galleries with thousands of images/videos without excessive DOM node overhead or memory consumption? Evaluate lightweight virtual scrolling approaches (e.g., svelte-virtual or simple intersection-observer based windowing).
3. **PWA & Offline Asset Strategy:** What caching strategy should the service worker adopt to cache app shell, icons, and recent thumbnails on client devices without exhausting low-end mobile phone storage?
4. **Touch Gestures & Media Viewer:** What minimal library or vanilla approach should power the lightbox viewer (swipe to navigate, pinch to zoom, inline HTML5 video playback)?
