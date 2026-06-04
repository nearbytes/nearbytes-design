# Projection engine — v1

Status: design approved; implementation in `nearbytes-log` (cores),
`nearbytes-files` + `nearbytes-chat` (projectors), `nearbytes-engine` (wiring).

## Problem

"Replay" of the event log into materialized application state was ad-hoc and
scattered:

- `nearbytes-files` kept an in-memory `FileReplayContext` cache with bespoke
  seed / tail / `hasOrderedPrefix` logic spread over `fileService.ts` and
  `fileEmit.ts`.
- `nearbytes-chat` had **no** cache: every `readChatTimeline` re-listed and
  re-hydrated **every** event of the channel.
- `nearbytes-engine` re-implemented divergent inbound-sync policies
  (`runtime.ts` blunt reload-all, `chatPush.ts`, app `refreshActive` full
  reread), and the materialized state never persisted, so every process boot did
  a **full replay** of every opened volume.

We want a single, modular, state-of-the-art replay architecture where new events
are **always processed incrementally**, materialized state **persists** across
restarts, and full replay happens only when a volume is first opened on a machine
with no persisted state.

## Solution: two shared cores + thin per-protocol projectors

```
nearbytes-log                       nearbytes-files / nearbytes-chat      nearbytes-engine
─────────────                       ───────────────────────────────      ────────────────
Core 1: log router                  filesProjector  (ordered)            dumb wiring:
  subscribe(filter, sink)           chatProjector   (sorted/append)        opens sqlite stores
  push on storeEvent + sync recv    reduce + reorder + key + codec         boot bucket-ingest
                                                                           exposes files/chat API
Core 2: projection engine
  createProjection(log, channel,
    crypto, projector, store)
  ingest(entries) push/boot
  snapshot ladder, persistence
  MaterializedStore (sqlite / mem)
```

### Design principle: ordering is the protocol's, never the log's or engine's

The projection engine is **order-agnostic**. It never sorts and never assumes a
total order exists. A *total order* is one projector's policy: `nearbytes-files`
needs the causal topological order (FILES v0.5), `nearbytes-chat` opts into a
cheap `(publishedAt, eventHash)` order, and a purely commutative protocol would
use the `appendOrder` default and pay nothing. Ordering is expressed by the
projector's `reorder` callback; the log only routes; the engine only folds,
snapshots, persists, and — when `reorder` reports an out-of-order arrival —
replays from the nearest snapshot.

## Core 1 — log router (`nearbytes-log`)

The log is already the single persistence choke point (local `storeEvent`, and
`nearbytes-sync` reception both persist through it). It gains a push router:

```ts
subscribe(filter: { channel?: PublicKeyHex; protocols?: Set<string> },
          sink: (entries: EventLogEntry[]) => void): () => void;
```

`storeEvent` and the sync receive sink notify the router after a successful
persist. The router is key-agnostic: it pushes raw entries; payload hydration
(decrypt) is done by the projection core, which holds the channel key. A pull
`consume(predicate)` API was rejected because polling reintroduces the
non-incremental scans we are removing.

## Core 2 — projection engine (`nearbytes-log/src/projection/`)

```ts
interface Projector<TState, TKey> {
  readonly id: string;                         // 'nb.files.v0.5' | 'nb.chat.v1'
  initial(): TState;
  serialize(state: TState): Uint8Array;        // live + snapshot persistence
  deserialize(bytes: Uint8Array): TState;
  key(entry: EventLogEntry): TKey;             // compact, serializable order key
  reorder(prevKeys: readonly TKey[], newKeys: readonly TKey[]):
    { keys: TKey[]; insertAt: number };        // the ordering policy
  reduce(base: TState, orderedTail: readonly EventLogEntry[]): TState; // the fold
}
```

A projector is two callbacks (`reorder` + `reduce`) plus a key extractor and a
state codec. The engine persists the compact `TKey[]` **order index** (files:
`{hash, parent, ts, isFileVerb}`; chat: `{hash, ts}`) so insertion is computed
without rehydrating payloads; full payloads are hydrated only for the bounded
tail being folded.

`createProjection(log, channel, crypto, projector, store)`:

1. On create — load persisted order index + live state + snapshot metas; subscribe
   to the router for `channel`.
2. `ingest(entries)` (push sink **and** boot path) — dedupe;
   `{ keys, insertAt } = reorder(prevKeys, newKeys)`.
   - `insertAt === liveVersion` → **forward fold** `reduce(live, hydrate(tail))`
     (the append / common case).
   - otherwise → load nearest snapshot with `pos ≤ insertAt` (else `initial()`),
     hydrate the ordered suffix, `reduce(base, suffix)`. The bounded suffix is the
     only thing rehydrated. (Files-style out-of-order arrival.)
3. Persist order index + live state; maybe write a snapshot per ladder; prune per
   retention; emit a change event for UI consumers.
4. `state()` is O(1) warm.

### Logarithmic snapshot ladder (`snapshots.ts`)

Snapshots are keyed by order position + wall-clock. Retention keeps at most one
snapshot per **day** for recent days, collapsing to one per **week** further back
and one per **month** oldest, always keeping the latest. Nearest lookup is
`max pos ≤ target`. This bounds replay-from-snapshot to logarithmic snapshot
count while keeping recent inserts cheap.

### Persistence (`store.ts`, `sqliteStore.ts`)

`MaterializedStore` is a pluggable interface (order-index blob, live-state blob,
snapshot blobs+meta, and a small KV for chat's last-seen / reply-to). Two
implementations:

- `createInMemoryMaterializedStore()` — tests and `browser.ts` builds.
- `createSqliteMaterializedStore(path)` — Node, built on the built-in
  `node:sqlite` (Node 22; zero native dependency). Node-only, never exported from
  `browser.ts`. Tables namespaced by `(projectorId, channelHex)`: `order_index`,
  `live_state`, `snapshots`, `meta`.

Files persists to `files.sqlite3`, chat to `chat.sqlite3`, under
`dataDir/.nearbytes/`. A future revision will move the store behind an internal
NearBytes volume protocol; only `sqliteStore.ts` changes.

## Projectors

- **files** (`nearbytes-files/src/filesProjector.ts`, ordered): `reorder` is the
  causal topo merge (reuse `orderEventLogEntries` / `hasOrderedPrefix` from
  `fileLogEntries.ts`) returning the earliest changed index as `insertAt`;
  `reduce` is `materializeIncremental` (tail) / `runMaterialization` (rebuild);
  state = `MaterializedFileSystem` + `liveEncryptedKeys` + `observedHead`.
  `fileService.ts` is rebuilt on a projection instance, dropping `replayCache`,
  `loadFileReplayContext` seed/tail, `applyInboundEvent`, `markReplayStale`,
  `extendFileReplayContext`. The blunt `inboundEventReadyToMaterialize` gating is
  removed — materialize on arrival (partial views allowed; byte reads still gate
  on blob presence).
- **chat** (`nearbytes-chat/src/chatProjector.ts`): sorted by `(publishedAt,
  eventHash)`; `reduce` parses/verifies/inserts (reuse `projectChatTimeline`
  codecs); KV holds the last-seen event hash (chat "blockref") and reply target.
  `chatService.ts` exposes engine-backed `timeline`, `publish`, `ingest`.

## Engine wiring (`nearbytes-engine`)

`runtime.ts` opens the two sqlite stores, builds the files engine + chat service,
and wires the log router so both receive pushes. Boot **bucket-loads**: read
events newer than each channel's persisted order index, bucket by channel, and
ingest volume-by-volume. `chatPush.ts` and the blunt `attachSyncInboundRefresh`
reload-all are deleted; one push policy is shared by CLI and app. `engine.ts`
becomes thin and **exposes** the files/chat APIs instead of re-deriving
`readChatTimeline` / `refreshActive`; `ui-state.json` (session UX) stays.

## Incremental guarantees

- Local write: `storeEvent` → router → forward-fold → persist.
- Sync inbound: reception persists → router → forward-fold (snapshot-bounded
  replay only when a projector reports an earlier `insertAt`).
- Boot / warm: live state loaded from sqlite; only newer events ingested.
- Cold / new volume: no persisted state → one full replay → snapshot + persist.

## Related

- Spec: `nearbytes-specs/storage/projection-engine-v1.md` (normative).
- Supersedes the replay-cache description in `file-events-v0.5.md` §11 and the
  full-reload chat replay in `chat-v1.md` §5.
- Resolves `nearbytes-specs/WIP.md` §§2–8.
