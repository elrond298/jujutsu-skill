# Workspaces

Use this guide when you need a second working copy of the same repo — e.g. running a long build or test in one workspace while editing in another.

A **workspace** is a working copy plus its associated repo. One repo can have multiple workspaces — each with its own working directory and working-copy commit (`@`) — all sharing the same commits, operations, and bookmarks. This is jj's equivalent of `git worktree`.

Workspaces are a rarely-needed feature; consult the [official docs](https://docs.jj-vcs.dev/latest/working-copy/#workspaces) for anything beyond the basics below.

## Common commands

```bash
# Create a new workspace (defaults: name = basename of path, parent = current @'s parent)
jj workspace add ../my-tests
jj workspace add --name tests -r <change-id> ../my-tests   # explicit name and base

# Inspect
jj --no-pager workspace list
jj workspace root [--name <ws>]

# Remove (does NOT delete files on disk — rm the directory separately)
jj workspace forget [<ws>]

# Rename current workspace
jj workspace rename <new-name>
```

In `jj log`, each workspace's `@` appears as `<workspace-name>@`.

## Key semantics

- **Isolation by default.** `jj workspace add` gives the new workspace its own fresh empty commit; workspaces don't start out sharing `@`, and on-disk files are never live-mirrored between them.
- **Propagation at command boundaries.** Each jj command snapshots the current workspace's files and reads the op log, so it sees commits/bookmarks made by other workspaces. There is no filesystem watcher.
- **Stale working copy.** If another workspace rewrites this workspace's `@` (e.g. via `jj squash`, `rebase`, `abandon`), jj refuses commands here until you run `jj workspace update-stale`. Same recovery path if a command was interrupted mid-update.
- **Shared `@` is sharp-edged.** `jj edit <id>` lets two workspaces point at the same change without warning. When one mutates it, the other goes stale; if the stale one had un-snapshotted edits, `update-stale` preserves them as a **divergent commit** (same change ID, shown as `xyz??` in `jj log`) that you must resolve. Avoid sharing `@` unless both workspaces are read-only.

## Agent guidance

- Always pass `--no-pager` to `jj workspace list`.
- Don't `jj edit` a change another workspace already has as its `@` — main cause of accidental divergence.
- Don't `rm -rf` a workspace directory without also running `jj workspace forget <name>`.