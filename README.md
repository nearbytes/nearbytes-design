# nearbytes-design

Design documents for Nearbytes modules.

Naming convention: design docs use the same filename/version as their matching
spec documents (for example `webdav-v1.md`, `file-events-v0.5.md`).

## Documents

| Document | Topic |
|---|---|
| [app-shell-v1.md](design/app-shell-v1.md) | Three-layer app shell: widgets / components / Electron; AppState; NearbytesAdapter; build pipeline |
| [projection-engine-v1.md](design/projection-engine-v1.md) | Log router + order-agnostic projection engine; per-protocol projectors; logarithmic snapshots; SQLite persistence |
| [file-events-v0.5.md](design/file-events-v0.5.md) | File event model and in-memory replay |
| [volume-session-v1.md](design/volume-session-v1.md) | Volume register / use / forget; persistence |
| [webdav-v1.md](design/webdav-v1.md) | WebDAV server v1 |
| [webdav-v2.md](design/webdav-v2.md) | WebDAV server v2 |
