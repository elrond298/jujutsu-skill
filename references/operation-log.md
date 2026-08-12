# Recovering with the operation log

The commit graph records project history. The operation log records what Jujutsu did to the repository itself. Use it when a rebase, squash, abandon, bookmark move, Git import, or concurrent command left the repository in an unexpected state.

## Learn the two histories

`jj log` shows commits. `jj op log` shows repository operations. Each operation stores a view containing visible heads, bookmarks, tags, Git refs, and every workspace's working-copy commit.

`jj evolog` is different again: it shows old commit versions for one change ID. Use the operation log for repository-wide recovery and the evolution log for the history of one mutable change.

## Inspect before changing anything

```bash
jj --no-pager op log
jj --no-pager op diff --operation OPERATION --patch --git
jj --no-pager --at-op=OPERATION log
jj --no-pager --at-op=OPERATION st
```

`--at-op` loads the repository view recorded after that operation. It skips automatic working-copy snapshotting, which makes it suitable for read-only diagnosis. In that view, `@` means the working-copy commit recorded at that time.

## Choose the narrowest recovery

Use `jj undo` when the last operation is wrong:

```bash
jj undo
jj st
```

Use `jj redo` if that undo was itself a mistake.

Use `jj op revert OPERATION` when an older operation is wrong but later work should remain. Jujutsu creates a new operation that applies the inverse of the selected one.

Use `jj op restore OPERATION` only when the entire repository view should return to that point. This moves bookmarks, heads, and workspace commit pointers back to the stored view, so inspect it with `--at-op` first.

A recovery sequence should therefore be:

1. Read `jj op log` and identify the operation that introduced the problem.
2. Inspect the old and current views with `--at-op` and `op diff`.
3. Pick `undo`, `op revert`, or `op restore` based on how much later work must survive.
4. Run `jj st`, `jj log`, and `jj op log` to verify the result.

## Colocated Git operations

A mutating Git command is imported by the next `jj` command and appears in the operation log as an import of Git refs. This gives you a Jujutsu recovery point for local repository state even though Git performed the mutation.

The operation log does not reverse external side effects. For example, restoring an operation cannot retract a push from a remote server.

## Concurrent operations

Jujutsu can record operations from concurrent processes without corrupting the repository. If two processes rewrite the same change, integration may expose divergent changes or bookmark conflicts. The operation log tells you which processes and commands created the competing states; [conflicts.md](conflicts.md) explains how to resolve the result.

## Sources

This guide combines the recovery flow demonstrated in the official [tutorial](https://docs.jj-vcs.dev/latest/tutorial/) with the data model from the official [operation log guide](https://docs.jj-vcs.dev/latest/operation-log/).