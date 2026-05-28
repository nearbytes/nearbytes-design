# WebDAV v2 — one mount, many volumes, time-travel read

Status: specified; implementation pending in `nearbytes-files`.

## Product

1. Mount once: `https://127.0.0.1:9843/`.
2. Root lists registered volume folders (`/<prefix>/`).
3. Global HTTP Basic, tied to **active sync profile**; re-auth on `profile use`.
4. WebDAV inactive until a profile is active in the REPL.
5. REPL `timeline goto` moves a cursor; WebDAV shows that prefix snapshot
   (read-only). Live head = read-write on WebDAV and normal REPL writes.

Writes never fork history at the cursor — they always append at the true log tip.

## Why global auth

Per-volume Basic forced the password on every mount. Registering volumes in
session retention lets WebDAV serve all open volumes after one authentication,
while secrets are entered only on `volume add` (or config bootstrap).

## Finder behavior

When the cursor steps backward, PROPFIND/GET reflect an older materialization:
files appear, disappear, or change size. That is expected read-only projection,
not mutation of the channel. Finder may cache aggressively; ETag SHOULD reflect
cursor + event head.

## Implementation notes

- Reuse `FileService` in-memory replay; add `replayContextThrough(cursorHash)`.
- Handler: route `/<volumeName>/` via session map; gate writes on cursor === head.
- Invalidate WebDAV auth cache on profile switch.

Normative: `nearbytes-specs/application/webdav-v2.md`.
