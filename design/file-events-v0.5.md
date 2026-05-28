# FILES v0.5 Design Notes

Design companion for `nearbytes-specs/application/file-events-v0.5.md` and
`nearbytes-files/docs/specs/file-events-v0.5.md`.

## Intent

- Keep event payload verbs unchanged (`CREATE_FILE | MKDIR | DELETE | RENAME`).
- Encode causal application dependencies in envelope `blockRefs`.
- Keep replay deterministic via observed-log-head parent edge + ready-set
  timestamp/hash tiebreak.

## Runtime Model

### Canonical replay

Full replay from the causally ordered hydrated log is the semantic ground truth.

### Incremental replay

When new events extend the previous ordered prefix, the materializer MAY apply
only the tail entries to an existing `MaterializedFileSystem`. This MUST yield
the same live state as full replay.

### In-memory channel log

`nearbytes-files` does **not** treat the on-disk log as the hot path for every
operation:

1. **Hydrate once** — `loadFileReplayContext` reads storage, decrypts payloads,
   verifies signatures, and builds `orderedEntries`.
2. **Serve from RAM** — `getReplayContext` returns the cached snapshot for reads.
3. **Append locally** — `emitFileEvent` returns a hydrated `EventLogEntry`;
   `extendFileReplayContext` appends it without re-listing the whole channel.
4. **Merge externally** — `markReplayStale` + incremental disk merge when
   `dataDir` changes under a running process.

The log API (`nearbytes-log`) remains append-only on disk. The FILES layer adds
a session cache above it.

## Interop

- `blockRefs` remain untyped in cleartext.
- Sync/storage must not assume every ref is a block hash.
- Typed role metadata stays encrypted.

## WebDAV / CLI consumers

Both WebDAV and `nbf timeline` consume the same `FileReplayContext`. There is
no separate WebDAV snapshot store. Performance fixes belong in the shared replay
cache, not in transport-specific caching.
