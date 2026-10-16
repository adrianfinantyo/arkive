# Arkive

An ultra-lightweight, self-hosted file management and media sharing platform optimized for legacy hardware, drive longevity, and smooth mobile gallery viewing.

## Language

**Admin**:
A user with full system control, global configuration, user management, and visibility over all hosted files and shares.
_Avoid_: Superuser, root

**Uploader**:
An authenticated user who uploads, organizes, and owns their files and folders, and controls their share permissions.
_Avoid_: Editor, contributor, owner

**Viewer**:
A read-only user accessing media galleries and downloading content via explicit user access or public links.
_Avoid_: Guest, consumer

**Group**:
A named collection of users managed by an Admin, allowing members to access shared folders and media.
_Avoid_: Team, workspace, organization, role

**Raw Media Store**:
The primary bulk storage volume hosted on the secondary hard drive (`/mnt/storage`) holding original full-resolution files.
_Avoid_: Bulk drive, archive disk

**Preview Cache**:
The SSD-backed cache (`/`) holding lightweight WebP thumbnails and extracted video keyframes to serve galleries without waking the storage drive.
_Avoid_: Thumbnail folder, temp store

**Share Link**:
A dynamic public or restricted link providing access to a file or folder, resolving contents in real-time on request.
_Avoid_: Public URL, share token

**Drive Spin-Down**:
Operating state where the secondary storage drive enters low-power sleep during idle periods, insulated from requests by the Preview Cache.
_Avoid_: Disk hibernate, standby mode

**Trash Bin**:
A temporary holding area for soft-deleted media and albums, retained for a configured duration ($X$ days) prior to irrevocable destruction.
_Avoid_: Recycle bin, archive holding

**Permanent Deletion**:
The irreversible destruction of a media object, atomically cascading across the database metadata, the Raw Media Store on `/mnt/storage`, and the Preview Cache on SSD.
_Avoid_: Hard purge, shred, obliterate
