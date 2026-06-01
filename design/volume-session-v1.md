# Volume session — register, use, forget

Status: specified; implemented in `nearbytes-files` REPL.

## Problem

Today `nbf` keeps open volumes only in RAM. WebDAV v1 required one Finder mount
per volume and typed the secret on every `open`. We want:

- persist **which volumes are open** under `dataDir` (`0600`);
- register secrets once, then `use` by short name (like sync profiles);
- never expose `channels/<pubkey>/` without the matching secret.

## Model

- **Registered** — name + secret in `<dataDir>/.nearbytes/volume-session.json`.
- **Open** — loaded in the REPL process (`volume add` / `volume use`; on REPL start
  only the **active** volume is reopened).
- **Active** — one volume for FTP-style commands and timeline cursor.

Commands: `volume add`, `volume use`, `volume forget`, `volume list`; aliases
`use`, `forget`, `volumes`. `open <secret>` remains convenience sugar.

Directory name for WebDAV and UX = secret prefix (`test2` from `test2:pass`).

## Timeline coupling

Cursor is per active volume, resets on `volume use`, not persisted.

Normative: `nearbytes-specs/application/volume-session-v1.md`.
