# App-shell architecture — v1

Status: implemented; shipped in `nearbytes-app`, `nearbytes-components`, `nearbytes-widgets`.

## Problem

A desktop NearBytes client needs a Finder-like file manager (breadcrumbs, folder
navigation, inspector, version history) plus a hub chat panel and sidebar
management of profiles, hubs and friends — all with state-of-the-art non-modal
UX and full feature parity with the `nbf` CLI.  The same UI must be iterable
purely as a design artefact, with no Electron process running, and must be
deployable identically on every machine from a clean install.

## Solution: three-layer shell

```
nearbytes-widgets          nearbytes-components            nearbytes-app (Electron)
────────────────           ────────────────────            ────────────────────────
Design tokens              AppState ($state tree)          src/main/
Tailwind v4 theme          NearbytesAdapter (interface)      service.ts  ← runtime
shadcn-svelte              AppShell / FinderShell          src/preload/
  primitives:              FileBrowser (breadcrumbs)         index.ts    ← bridge
  Panel / List /           FileInspector (rename/meta)     src/renderer/
  ChatBubble /             ChatPane                          App.svelte  ← 12 lines
  ManagedList /            SourcesPanel                      ipcAdapter.ts
  InsetGroup / …           IdentityFooter                    hydrate.ts
```

Each layer has a strict contract with the one above; nothing leaks downward.

## Layer 1 — nearbytes-widgets

Pure design-system package. No NearBytes domain knowledge.

- **Tokens** (`src/lib/styles/tokens.css`): macOS dark-mode palette as Tailwind
  v4 `@theme` variables (`--color-nb-bg`, `--color-nb-accent`, `--radius-nb`,
  `--font-sans`, …).
- **shadcn-svelte primitives**: Button, Input, Label, ScrollArea, DropdownMenu,
  ContextMenu, AlertDialog, Resizable, Tooltip, Card, Badge, Alert, Separator,
  Avatar.
- **NearBytes widgets**: Panel, PanelTitle, InsetGroup, List, ListItem, MenuEntry,
  MenuItemLabel, EmptyState, StatusIndicator, FilePreview, Icon, ChatBubble,
  ChatComposer, ConfigSection, ConfigField, ConfigToggle.

No imports from `nearbytes-*` domain packages; no Node/Electron.  Depends only
on `bits-ui`, `paneforge`, `clsx`, `tailwind-merge`, `tailwind-variants` from npm.

## Layer 2 — nearbytes-components

App-level component library.  Composes widgets into the complete NearBytes UX.
Contains **all** layout, all UI logic, and all UX policy.  No Node, no Electron.

### State — `AppState`

Single Svelte 5 deep-reactive `$state` tree.  Defined in
`src/lib/stores/appState.svelte.ts`.  Sub-records are passed to components **by
reference**; mutations propagate via Svelte's deep-proxy reactivity with no
explicit subscriptions.

```ts
interface AppState {
  status:        AppStatus           // { text: string, kind: StatusKind }
  identity:      Identity            // { publicKey: string|null, peerId: string|null }

  profiles:      ProfileConfig[]     // registered sync profiles
  activeProfile: string | null

  hubs:          VolumeConfig[]      // registered hubs (volumes)
  activeHub:     string | null

  friends:       string[]            // followed peers (public-key hex)

  files: {
    items:         FileMetadata[]
    directories:   DirectoryMetadata[]
    cwd:           string            // current folder path within active hub
    selectedPath:  string | null
  }

  chat: {
    items:  ChatTimelineItem[]
    draft:  string
  }
}
```

All domain record types (`FileMetadata`, `ChatTimelineItem`, `ProfileConfig`, …)
are imported **type-only** from the protocol packages.  No runtime imports.

### Adapter boundary — `NearbytesAdapter`

Pure TypeScript interface in `src/lib/adapter.ts`.  Declares every capability the
UI needs as async methods, grouped by API surface:

```
profile  — list / add / use / remove / update / reorder / publish / publicKey
hub      — list / add / use / forget / update / reorder
file     — list / add / get / remove / mkdir / rename / timeline / openExternally
chat     — read / say
friend   — list / add / remove / reorder
service  — status() / whoami() / peers()
events   — onStatus / onActiveVolume / onChat  (push, return unsubscribe fn)
```

Two implementations exist and are interchangeable:

| Context | Implementation |
|---|---|
| Electron renderer | `nearbytes-app/src/renderer/lib/ipcAdapter.ts` — thin wrapper around `window.nb.invoke` |
| Preview / design harness | `nearbytes-components/preview/mockAdapter.ts` — in-memory model |

Components never reference either concrete implementation; they call
`useAdapter()` (Svelte context) and get the interface.  This is what makes the
design loop described below work.

### Svelte context injection

Two context keys provide the state tree and adapter downward through the
component tree without prop-drilling:

```ts
provideAppState(state)  // called once in App.svelte / Harness.svelte
provideAdapter(adapter) // called once in App.svelte / Harness.svelte

useAppState()           // any child component — returns AppState
useAdapter()            // any child component — returns NearbytesAdapter
```

### Component tree

```
AppShell
├── titlebar (logo, ProfileSelector dropdown)
├── Resizable.PaneGroup (horizontal)
│   ├── FinderShell
│   │   ├── Resizable.PaneGroup (horizontal)
│   │   │   ├── SourcesPanel
│   │   │   │   ├── ManagedList (Profiles)
│   │   │   │   ├── ManagedList (Hubs / volumes)
│   │   │   │   ├── ManagedList (Friends)
│   │   │   │   └── IdentityFooter (copy key / publish)
│   │   │   ├── FileBrowser
│   │   │   │   ├── Breadcrumbs
│   │   │   │   ├── toolbar (search / new-folder / upload)
│   │   │   │   └── FileEntryRow × n (dirs first, then files)
│   │   │   └── FileInspector
│   │   │       ├── hero (icon, inline rename)
│   │   │       ├── action buttons (Open / Remove)
│   │   │       ├── metadata dl (Where / Size / Kind / Created / Blob)
│   │   │       └── VersionHistory (timeline)
│   └── ChatPane
│       ├── ChatMessageView × n (grouped, own = right/blue)
│       └── ChatComposer
└── StatusBar (status indicator, text, peer fingerprint)
```

### UX policy — non-modal everywhere

All destructive and configuration actions are embedded in the UI; no `<dialog>`
or `AlertDialog` is used for routine operations:

- **Delete confirm** — row expands in-place to "Delete X? Cancel / Delete" strip.
- **Add / edit** — inline form expands below the section header; Cancel collapses it.
- **Rename** — filename in inspector becomes an `<input>` on click; Escape cancels.
- **New folder** — inline form above the file list; Escape cancels.
- **Publish identity** — collapsible panel in the sidebar footer.

### Visual design — preview harness

`nearbytes-components/preview/` is a standalone Vite + Svelte + Tailwind app
that mounts the full `AppShell` against the mock adapter with seeded realistic
data.  Run with `yarn dev` inside `nearbytes-components`; serves at
`http://localhost:5199`.  No Electron, no GitHub packages, instant HMR.

This is the primary design-iteration environment.  Changes push to GitHub,
then `yarn refresh` in `nearbytes-app` picks them up.

## Layer 3 — nearbytes-app (Electron)

The app renderer is **5 files** of pure bootstrapping:

| File | Role |
|---|---|
| `src/renderer/App.svelte` | Creates `AppState`; creates `IpcAdapter`; provides both via context; mounts `AppShell`. 12 lines. |
| `src/renderer/main.ts` | Calls Svelte `mount(App, {target})`. 4 lines. |
| `src/renderer/app.css` | Imports Tailwind + design tokens; sets `html/body/#app { height:100% }`. |
| `src/renderer/lib/hydrate.ts` | Async initial fill: calls adapter to populate all `AppState` fields; wires the 3 push-event listeners (`onStatus`, `onActiveVolume`, `onChat`). |
| `src/renderer/lib/ipcAdapter.ts` | Maps every `NearbytesAdapter` method to `window.nb.invoke({api, method, args})`. |

No layout, no components, no CSS classes.  The renderer contains zero UI.

The main process (`src/main/service.ts`) runs the NearBytes runtime:
`createFilesystemSkeletonFromConfig` → file service → sync.  This is identical
to the CLI bootstrap.  IPC handlers in `src/main/ipc.ts` dispatch to the same
service layer.

### IPC contract

Defined in `src/shared/ipc.ts` — shared by main, preload and renderer.

```ts
// renderer → main (request/response)
invoke: { api: 'profile'|'hub'|'file'|'chat'|'friend'|'service', method: string, args: unknown[] }

// main → renderer (push)
PushEvent:
  | { channel: 'status',  payload: SyncStatus }
  | { channel: 'volume',  payload: VolumeView }
  | { channel: 'chat',    payload: ChatTimelineItem[] }
```

### Build pipeline — GitHub-only, fully automated

All NearBytes packages are consumed as `github:nearbytes/<pkg>` and built by
their own `prepare`/`prepack` script during install.  There are **no local-sibling
source aliases, no file: deps, no monorepo workspace**.

`scripts/refresh.mjs` re-resolves every `nearbytes-*` dep to the latest GitHub
HEAD (`yarn up nearbytes-<pkg>@github:nearbytes/nearbytes-<pkg>` for each),
then deletes `node_modules/.vite` to prevent stale pre-bundle serving.

```
yarn refresh         re-fetch all nearbytes-* from GitHub HEAD + clear Vite cache
yarn dev             refresh, then electron-vite dev
yarn build           refresh, then electron-vite build → out/
yarn package         refresh, build, then electron-builder
yarn dev:fast        electron-vite dev (skip refresh)
yarn build:fast      electron-vite build (skip refresh)
```

## Data-flow summary

```
Electron main process
  └── NearBytes runtime (skeleton / files / sync / chat)
        │  events: status, volume snapshot, chat snapshot
        │  IPC push: nb:event → preload → renderer window.nb.on()
        ↓
  hydrate.ts  (initial fill on mount)
  onStatus / onActiveVolume / onChat  (live push)
        │  mutate AppState in place
        ↓
  AppState ($state deep-reactive proxy)
        │  Svelte reads every field used in templates; re-renders on mutation
        ↓
  Component tree (AppShell → … → leaf widgets)
        │  user interaction (click, drag, type)
        ↓
  useAdapter().profile|hub|file|chat|friend.*()
        │  IPC invoke: nb:invoke → preload → main ipc.ts → service
        ↓
  NearBytes runtime (side-effect: write to skeleton / network)
        │  triggers next push event cycle
        └──────────────────────────────────────────────────────► (loop)
```

## Design iteration workflow

```
1. Edit  nearbytes-components/src/lib/**/*.svelte  (HMR at localhost:5199)
2. Verify in preview harness (no Electron needed)
3. Commit + push nearbytes-components to GitHub
4. In nearbytes-app:  yarn refresh   (fetches latest, clears cache)
5. yarn dev:fast      (open Electron with the new UI)
```

If the widgets design-system also changed, push `nearbytes-widgets` first (step 3a),
then update the widgets ref in `nearbytes-components` (`yarn up
nearbytes-widgets@github:nearbytes/nearbytes-widgets`), then push components.
