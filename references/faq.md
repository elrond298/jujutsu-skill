# Jujutsu troubleshooting guide

When something looks wrong, inspect before changing it:

```bash
jj st
jj --no-pager log -r '..'
jj --no-pager op log
```

These three views answer different questions: what is in the working copy, where visible commits and bookmarks point, and which repository operation produced the state.

## Why did my bookmark stay behind?

Jujutsu has no current bookmark. Creating a child with `jj new` or `jj commit` does not move any bookmark.

```bash
jj bookmark move NAME --to REV
```

Use `--allow-backwards` when the move is not a simple fast-forward.

## Why did push say "Nothing changed"?

`jj git push --all` pushes bookmarks and tags, not every visible commit. Either move or create a bookmark, or let Jujutsu create a generated bookmark:

```bash
jj git push -c REV
```

## Where did my commit go?

The default log is intentionally selective. Search all visible commits first:

```bash
jj --no-pager log -r 'all()'
```

If you know the commit ID, query it directly. An abandoned commit or an obsolete version of a rewritten change may be hidden, but it remains addressable by commit ID. `jj new COMMIT_ID` makes it visible again.

Use `connected(REVSET)` when the log says revisions were elided and you need the commits between them.

## Where are automatic working-copy saves?

Every `jj` command snapshots the working-copy commit when needed. Its commit ID changes, while its change ID stays stable. Those versions are not a stack in the normal log.

```bash
jj --no-pager evolog -p -r @
```

Use a hidden commit ID from the evolution log to inspect or recover an earlier version.

## How do I save unfinished work?

It is already saved in the working-copy commit. Run `jj new` when you want a checkpoint, then later combine the draft commits with `jj squash`.

Files that should never be tracked belong in ignore rules. To limit automatic tracking of new files, configure `snapshot.auto-track`; already tracked files still get snapshotted.

## How do I split or amend part of a change?

For a human with an interactive editor:

```bash
jj split -i
jj commit -i
jj squash -i
```

For an agent, avoid interactive commands. Split by path with `jj commit -m "Message" PATH...` while work is still in `@`, or use `jj split -m "Message" PATH...` for an existing revision.

## I edited the wrong change. What now?

If the mistake was the last repository operation, use `jj undo`. Otherwise:

1. Find the last good commit ID with `jj evolog --patch --git` or a remote bookmark.
2. Start a new child at that good version with `jj new GOOD_COMMIT_ID`.
3. Restore the desired content from the bad version with `jj restore --from BAD_COMMIT_ID`.
4. Move any bookmark from the bad commit to the good one.
5. Abandon the bad commit after verifying both diffs.

This can create temporary divergence because both old versions become visible. Use commit IDs until the unwanted version is abandoned.

## How do I resume an earlier change?

Prefer a reviewable child:

```bash
jj new REV
# edit
jj --no-pager diff --git
jj squash -u
```

`jj edit REV` is shorter but amends the revision directly. Avoid it on conflicted revisions because marker edits are harder to review separately.

## Why is a merge commit empty?

Jujutsu computes a merge commit's change against the automatic merge of all parents. A clean merge adds nothing beyond that parent combination, so it is correctly marked empty. Conflict resolutions or extra edits make the merge non-empty.

To undo what one parent contributed, do not revert the empty merge itself. Create a child of the merge and restore the first parent's tree:

```bash
jj new MERGE
jj restore --from FIRST_PARENT
jj desc -m "Revert merge"
```

## Should I colocate with Git?

Colocation is a good default while learning and when tools expect `.git/`. Use Jujutsu for mutations and Git for inspection. Choose non-colocation only for a concrete reason, such as background Git writers, confused tools around conflicted commits, or costly imports in a repository with very many refs. See [colocated-repos.md](colocated-repos.md).

## Is this a file conflict, divergence, or bookmark conflict?

- A file conflict is stored in a commit and appears in `jj log -r 'conflicts()'`.
- Divergence means several visible commits share one change ID.
- A bookmark conflict means one bookmark points to several commits and is marked `??`.

Follow [conflicts.md](conflicts.md) because each state has a different fix.

## Why do Vite or Vitest and Jujutsu interfere?

Vite may watch `.jj/`, causing slow startup, command timeouts, or lock contention. Add `"**/.jj/**"` to `server.watch.ignored` in Vite configuration.

## CLI or library for an integration?

`jj-lib` avoids parsing command output but is not a stable API. The CLI is also versioned without a stability guarantee, but it works with custom Jujutsu binaries and backends. Choose based on that tradeoff and test against the versions you support.

## Source

The official [Jujutsu FAQ](https://docs.jj-vcs.dev/latest/FAQ/) contains the longer examples behind these diagnostic paths.