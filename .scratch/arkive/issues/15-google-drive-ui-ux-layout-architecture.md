# Google Drive-Style UI/UX & Layout Architecture

Type: grilling
Status: open
Blocked by: 03, 05, 10

## Question

How should Arkive design and organize its SvelteKit frontend layout, navigation hierarchy, list/grid dual view, right-side details inspector, and drag-and-drop interactions to deliver a Google Drive experience optimized for low client memory (<25MB)?

Specific decisions to resolve with the user:
1. **Three-Pane Layout Shell & Collapsible Sidebars:** How should the primary layout shell be structured (left navigation sidebar with "+ New" upload button, top search and status bar, main browsing area, and right collapsible metadata inspector)?
2. **Dual-Mode Browsing (Folder Hierarchy + Virtualized Photo Grid):** How should the interface bridge Google Drive folder navigation (breadcrumbs, subfolder chips) with the $O(1)$ uniform-row virtualized photo gallery from ADR-0003, and how is the List vs Grid view toggle managed?
3. **Right-Side Inspector & Metadata Drawer:** What information should the right-side inspector display (file details, EXIF camera metadata, storage tier badge showing SSD Preview vs HDD Raw Media, and Group sharing permissions)?
4. **Context Menus, Drag-and-Drop & Batch Actions:** How should multi-selection toolbars (Download Zip, Share, Move, Trash), right-click context menus, and drag-and-drop file ingestion into folders function across desktop and mobile?
