# beads-ui Architecture (v2)

Note (2025-10-26)

- beads-ui has migrated to a push‑only protocol for lists and details. The
  server no longer implements legacy read RPCs `list-issues` and `epic-status`.
  For the normative protocol reference, see `docs/protocol/issues-push-v2.md`.

This document describes the high‑level architecture of beads‑ui and the v2
push‑only data flow used between the browser SPA and the local Node.js server.

## Overview

- Local‑first single‑page app served by a localhost HTTP server
- WebSocket for data (request/response + server push events)
- Server bridges UI intents to the `bd` CLI and watches the active workspace's
  Beads storage location for changes

```
+--------------+          ws://127.0.0.1:PORT/ws          +--------------------+
|  Browser SPA | <--------------------------------------> | HTTP + WS Server   |
|  (ESM, DOM)  |   requests (JSON) / replies + events     |  (Node.js, ESM)    |
+--------------+                                          +---------+----------+
                                                               invokes | | watches
                                                                       v v
                                                               +--------+ +---------------------+
                                                               | bd CLI | | SQLite .beads/*.db  |
                                                               +---+----+ | or metadata-backed  |
                                                                   |      | .beads/ directory   |
                                                                   +----> +---------------------+
                                                                 reads/writes
```

## Components and Responsibilities

- UI (app/)
  - `app/main.js`: bootstraps shell, creates store/router, wires WS client,
    refreshes on push
  - Views: `app/views/list.js`, `app/views/detail.js` render issues and allow
    edits
  - Transport: `app/ws.js` persistent client with reconnect, correlation, and
    event dispatcher
  - Protocol: `app/protocol.js` shared message shapes, version, helpers, and
    type guards

- Server (server/)
  - Web: `server/app.js` (Express app), `server/index.js` (startup and wiring)
  - WebSocket: `server/ws.js` (attach server, parse, validate, dispatch
    handlers, broadcast events)
  - bd bridge: `server/bd.js` (spawn `bd`, set `BEADS_DB` for a discovered
    SQLite DB, JSON helpers)
  - Storage resolution/watch: `server/db.js` (resolve SQLite file or
    metadata-backed workspace directory), `server/watcher.js` (detect changes
    and schedule list refresh)
  - List adapters: `server/list-adapters.js` (map subscriptions, including
    `ready-issues`, to `bd` reads)
  - Config: `server/config.js` (bind to `127.0.0.1`, default port 3000)

## Data Flow

1. User action in the UI creates a request `{ id, type, payload }` via
   `app/ws.js`.
2. Server validates and maps the request to a `bd` command (no shell; args array
   only).
3. Server replies with `{ id, ok, type, payload }` or `{ id, ok:false, error }`.
4. The storage watcher schedules a debounced refresh for active list
   subscriptions; clients receive `snapshot`/`upsert`/`delete` envelopes.
   Mutations also trigger a watcher-or-timeout refresh gate.

## Protocol (v2.0.0)

Envelope shapes (see `app/protocol.md` and `docs/protocol/issues-push-v2.md`):

- Request: `{ id: string, type: string, payload?: any }`
- Reply:
  `{ id: string, ok: boolean, type: string, payload?: any, error?: { code: string, message: string, details?: any } }`

Push‑only subscriptions for lists and details:

- Client subscribes via `subscribe-list` with `{ id, type, params? }`.
- Server acks the subscription and immediately publishes a `snapshot` envelope
  with the full list for that subscription id followed by `upsert`/`delete`
  envelopes as data changes. Clients render from local per‑subscription stores.

Common message types (mutations only; list reads removed):

- `update-status` payload:
  `{ id: string, status: 'open'|'in_progress'|'closed' }`
- `edit-text` payload:
  `{ id: string, field: 'title'|'description'|'acceptance'|'notes'|'design', value: string }`
- `update-priority` payload: `{ id: string, priority: 0|1|2|3|4 }`
- `dep-add` payload: `{ a: string, b: string, view_id?: string }`
- `dep-remove` payload: `{ a: string, b: string, view_id?: string }`
- `create-issue` payload:
  `{ title: string, type?: 'bug'|'feature'|'task'|'epic'|'chore', priority?: 0|1|2|3|4, description?: string }`

Removed in v2:

- `list-issues`, `epic-status`, `subscribe-updates` and the legacy
  `issues-changed` event.

## UI → bd Command Mapping

- Lists and details: push‑only via `subscribe-list` (no list reads)
- Update status: `bd update <id> --status <open|in_progress|closed>`
- Update priority: `bd update <id> --priority <0..4>`
- Edit title: `bd update <id> --title <text>`
- Edit description: `bd update <id> --description <text>`
- Edit acceptance: `bd update <id> --acceptance-criteria <text>`
- Link dependency: `bd dep add <a> <b>` (a depends on b)
- Unlink dependency: `bd dep remove <a> <b>`
- Create issue: `bd create "title" -t <type> -p <prio> -d "desc"`

Rationale

- Use `--json` for read commands to ensure typed payloads.
- Avoid shell invocation; pass args array to `spawn` to prevent injection.
- List adapters pass the active workspace as `cwd` to `bd`. `server/bd.js` sets
  `BEADS_DB` for a discovered local SQLite file; it does not inject `--db` or
  force the global fallback path, so non-SQLite autodetection remains available.
  Mutation handlers currently omit `cwd` and run from the server process cwd.

## Issue Data Model (wire)

```ts
interface Issue {
  id: string;
  title?: string;
  description?: string;
  acceptance?: string;
  status?: 'open' | 'in_progress' | 'closed';
  priority?: 0 | 1 | 2 | 3 | 4;
  dependencies?: Array<{
    id: string;
    title?: string;
    status?: string;
    priority?: number;
    issue_type?: string;
  }>;
  dependents?: Array<{
    id: string;
    title?: string;
    status?: string;
    priority?: number;
    issue_type?: string;
  }>;
}
```

Notes

- Fields are optional to allow partial views and forward compatibility.
- Additional properties may appear; clients should ignore unknown keys.

## Error Model and Versioning

- Error object: `{ code: string, message: string, details?: any }`
- Common codes: `bad_request`, `not_found`, `bd_error`, `unknown_type`,
  `bad_json`

## Security and Local Boundaries

- Server binds to `127.0.0.1` by default to keep traffic local.
- Basic input validation at the WS boundary; unknown or malformed messages
  produce structured errors.
- No shell usage; `spawn` with args only; environment opt‑in via `BD_BIN`.

## Watcher Design

- The server resolves a local SQLite file or, when no SQLite file takes
  precedence, a workspace `.beads` directory containing `metadata.json` (see
  `docs/db-watching.md`).
- SQLite: watch the parent directory and filter by DB filename. Metadata-backed
  workspaces (for example Dolt): watch the `.beads` directory. The watcher
  debounces events and applies a cooldown; the server also coalesces refreshes.
- `set-workspace` re-resolves storage and rebinds the watcher when the resolved
  path changes. The server publishes subscription envelopes; the UI re-renders
  from per-subscription stores.

## Risks & Open Questions

- New-issue follow-up uses removed `list-issues` RPC
  - Owner: UI + Server
  - Next: Return the created ID in the `create-issue` reply and use it directly
    for navigation and labels; `app/views/new-issue-dialog.js` currently tries a
    legacy `list-issues` read after the create acknowledgement.
- Workspace switching does not redirect mutation commands
  - Owner: Server
  - Next: Pass the current workspace root as `cwd` to mutation and follow-up
    `bd` calls in `server/ws.js`; currently these calls use the process cwd,
    while subscription reads use the selected workspace.
- Large lists still require full `bd` reads on each refresh
  - Owner: Server + UI
  - Next: Measure large-workspace read and snapshot costs; consider bounded or
    paginated subscriptions. `upsert`/`delete` deltas reduce pushed updates but
    do not avoid fetching the list from `bd`.
- Cross-platform path resolution coverage
  - Owner: Server
  - Next: Add Windows/macOS path and filesystem-watcher integration coverage;
    current DB resolver tests primarily use POSIX-style paths.

---

For the normative protocol reference and unit tests, see `app/protocol.md` and
`app/protocol.test.js`.
