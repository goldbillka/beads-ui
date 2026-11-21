# DB Watching and Resolution

The server watches the active workspace's Beads storage location and schedules
refreshes of active list subscriptions. Clients receive
`snapshot`/`upsert`/`delete` envelopes for their active subscriptions.

## Resolution Order

`resolveWorkspaceDatabase` selects a watch target in this order:

1. Explicit `explicit_db` argument (available to the resolver/watcher; not
   supplied by normal server startup)
2. `BEADS_DB` environment variable
3. Nearest `.beads/*.db` walking up from the active workspace root (first
   filename alphabetically within the nearest directory)
4. Nearest `.beads/metadata.json` walking up from that root: watch its `.beads`
   directory for non-SQLite workspaces, for example Dolt
5. `~/.beads/default.db` fallback when neither a workspace DB nor metadata is
   found

The `bd` bridge sets `BEADS_DB` only for a discovered local SQLite DB. It does
**not** inject `--db` into CLI args or force the home fallback, allowing `bd` to
autodetect non-SQLite backends. An existing `BEADS_DB` environment value remains
in effect. Subscription reads run in the selected workspace; mutation handlers
currently run in the server process cwd even after `set-workspace` (see
`docs/architecture.md` risks). The watch target is a filesystem hint, not a
backend connection string.

## Watch and Refresh

- For a SQLite file, `fs.watch` observes its parent directory and filters events
  by filename. For a metadata-backed workspace, it watches the entire `.beads`
  directory.
- The watcher accepts change/rename events, debounces them (250 ms by default),
  and ignores events during a subsequent cooldown (1 s by default). The server
  coalesces refresh requests (75 ms at startup); mutations also arrange a
  watcher-or-timeout refresh. Refreshes re-read subscribed lists through `bd`
  and publish snapshots or deltas.
- On `set-workspace`, a changed resolved path causes the watcher to rebind and
  subscriptions to refresh. A mere change in storage location on disk without a
  workspace switch does not automatically re-resolve the target.

Examples:

| Workspace                                   | Watch target                                  | `bd` execution                                                    |
| ------------------------------------------- | --------------------------------------------- | ----------------------------------------------------------------- |
| `repo/.beads/beads.db`                      | Parent `repo/.beads/`, filtered to `beads.db` | Subscription reads run in `repo/` with `BEADS_DB` set to local DB |
| `repo/.beads/metadata.json`, no local `.db` | Entire `repo/.beads/`                         | Subscription reads run in `repo/` without forcing a SQLite path   |

## Behavior When Missing

If the resolved file does not exist (for example before `bd init`), the watcher
logs a debug hint and attempts to watch its parent directory. This succeeds only
if that directory exists; a failed `fs.watch` is logged, not retried
automatically. Initialize a workspace with `bd init`, or explicitly configure a
SQLite DB with, for example:

- `bd --db /path/to/file.db init`
- `export BEADS_DB=/path/to/file.db && bd init`
- `bd init` in the workspace (backend selected by `bd`)

If the watched parent existed, creation of the target file can be detected
without a restart. Otherwise restart the server or switch workspaces to rebind
the watcher after initialization.
