# WebDAV + FILES v0.5 Design

Status: implemented in `nearbytes-files`.

## Scope

- Implement WebDAV in `nearbytes-files`, served only on localhost over HTTPS.
- Preserve `FileService` as the single authoritative API for filesystem
  operations.
- Use visible event-envelope `blockRefs` as application-level dependencies.
- Keep cleartext refs untyped. Role-rich metadata belongs in ciphertext.
- FILES replay uses topological replay over the observed-log-head parent edge,
  with timestamps only as concurrency tiebreaks.
- Keep the **hydrated channel log in memory** per secret; do not reload the full
  channel from disk on every WebDAV request or CLI command.

## Product Behavior

- Mount URL: `https://host/<volume-name>/...`
- Basic auth:
  - `username = <volume-name>`
  - `password = <secret-part>`
  - effective channel secret = `username:password`
- Any volume name is accepted dynamically; no config pre-registration.
- Wrong credentials must not leak other volumes: empty history / decrypt-failure
  semantics only.
- HTTPS is mandatory for WebDAV credentials.
- Listener binds localhost only by default (`127.0.0.1`).
- WebDAV auto-starts in the REPL/default shell path, not one-shot commands.

### CLI

| Flag | Purpose |
|------|---------|
| `--webdav-port <n>` | HTTPS listen port (default `9843`) |
| `--debug [areas]` | `cli`, `webdav`, `timing` (comma-separated; all if omitted) |

No `NEARBYTES_*` environment variables for WebDAV debug or port.

## Normative Spec Targets

- `nearbytes-specs/application/blockrefs-v0.1.md`
- `nearbytes-specs/application/file-events-v0.5.md`
- `nearbytes-specs/application/webdav-v1.md`
- Package summaries: `nearbytes-files/docs/specs/*.md`

## FILES v0.5 Visible `blockRefs`

Normative clear ref order for FILES v0.5 producers:

1. `observedLogHead` — only topological replay parent; mandatory on non-empty channel.
2. direct predecessor refs — exact-path entry heads; not ordering edges.
3. previous-content refs — predecessor file blocks; not ordering edges.
4. introduced-content refs — new `CREATE_FILE` content.

Directory cascade is materializer semantics, not `blockRefs` expansion.

## Replay And Tiebreaking

Kahn topological sort over observed-log-head edges; among ready events choose
smallest FILES timestamp, then smallest event hash.

Valid target conflicts are latest-wins in canonical replay order.

## WebDAV Mapping

`src/webdav/handler.ts` → `FileService`:

| WebDAV | FileService |
|--------|-------------|
| `PROPFIND` | materialized snapshot (`getReplayContext`, optional `enrichSizes`) |
| `GET` / `HEAD` | snapshot metadata + `getFileByPath` |
| `PUT` | `addFile` |
| `DELETE` | `delete` |
| `MKCOL` | `mkdir` |
| `MOVE` | `rename` |

Writes do not fail on `If-Match`. Semantic parent = observed log head at commit.
`LOCK`/`UNLOCK` are non-blocking Finder shims.

### ETag and sizes

- File ETag = live FILES entry head (`fileOrigins` / `entryHeads`).
- Collection ETag = channel replay head (conservative).
- `PROPFIND` MUST report non-zero `getcontentlength` for non-empty files.
  Finder/macOS `webdavfs` uses this for display and copy.
- Plaintext size is not in `CREATE_FILE` payloads today; implementations use a
  blob-hash size cache on write and optional one-time decrypt for legacy files
  on WebDAV reads (`enrichSizes`).

## In-Memory Channel Replay (Core Design Decision)

The event log is **always in memory** for an open volume in a running `nbf`
process. Disk is the durability layer; RAM is the working set.

```
┌─────────────────────────────────────────────────────────┐
│ FileService replay cache (per channel secret)           │
│  • orderedEntries[]  (hydrated EventLogEntry)           │
│  • fs                (MaterializedFileSystem)           │
│  • liveEncryptedKeys (path → wrapped key)               │
│  • observedHead                                       │
└─────────────────────────────────────────────────────────┘
         ▲                              │
         │ extendFileReplayContext      │ getReplayContext (warm)
         │ (local emit)                 ▼
    emitFileEvent                   WebDAV / timeline / ls
         │
         ▼
   nearbytes-log (disk)
```

### Operations

| Trigger | Action |
|---------|--------|
| First `getReplayContext` | Cold load: list events, hydrate, verify, order, materialize |
| Repeat read | Return cache (0 disk reads for channel events) |
| Local `addFile` / `mkdir` / … | Append entry in RAM; incremental materialize |
| External sync (`nbsync`, other `nbf`) | `markReplayStale` → next read merges **new hashes only** |
| Import bundle (bulk emit) | `markReplayStale` after batch |

**Anti-pattern (removed):** calling `loadEventLog` + full verify on every write
because `invalidateReplayContext` forced a cold reload. That made `timeline` and
WebDAV feel like ~1 s per interaction on channels with hundreds of events.

### Correctness rule

In-memory append and incremental materialization MUST be observationally
equivalent to reloading the full channel from storage and running full canonical
replay.

## Materialization Entry Points

- `runMaterialization` — full replay from canonical entries
- `materializeIncremental` — append-only optimization
- `loadFileReplayContext` — cold / stale merge from disk
- `extendFileReplayContext` — pure RAM append after local emit
