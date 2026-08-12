# Anonymous branches, merging, and rebasing

Use this guide when the commit graph forks and you need to decide whether to keep the fork, merge it, or make it linear again.

## Think in graphs, not branch names

A branch exists whenever commits have diverged from a common parent. Jujutsu does not require a name for that topology. The description tells you what a change does, and its change ID gives you a stable way to address it after rewrites.

A bookmark is useful when another tool or person needs a durable name, especially when pushing to Git. It is not required for local work.

## Create sibling lines of work

Start both changes from the same base:

```bash
jj new main -m "Add better documentation"
# edit files
jj new main -m "Refactor printing"
# edit files
```

The two changes are siblings even though neither has a bookmark. Inspect the fork with:

```bash
jj --no-pager log
jj --no-pager log -r 'heads(all())'
```

Use change IDs from the log to move between lines of work. `jj new REV` starts a new child that you can later squash. `jj edit REV` amends that change directly.

## Choose merge or rebase

Merge when both parents should remain visible in history:

```bash
jj new DOCS_CHANGE CODE_CHANGE -m "Merge documentation and printing changes"
```

A merge is simply a new commit with multiple parents. A clean merge is often shown as empty because it adds no content beyond the automatic merge of those parents.

Rebase when one line of work should become a descendant of another:

```bash
jj rebase -r DOCS_CHANGE -o CODE_CHANGE
```

Choose what moves deliberately:

| Selection | What moves |
| --- | --- |
| `-r REV` | Only the selected revision |
| `-s REV` | The revision and all descendants |
| `-b REV` | The whole branch containing the revision, relative to the destination |

`-o DEST` selects the new parent. A rebase rewrites repository history, but it does not switch the working copy to the destination as Git often does. Change IDs remain stable while commit IDs change.

## A practical sequence

1. Run `jj --no-pager log` and identify the base and branch heads.
2. Decide whether the history should preserve both parents or become linear.
3. Use `jj new A B` for a merge, or `jj rebase` with the narrowest suitable selector.
4. Run `jj st` and `jj --no-pager log` after the operation.
5. If conflicts appear, leave the graph in place and follow [conflicts.md](conflicts.md). The operation has completed; there is no `--continue` step.
6. Use `jj undo` if the topology is not what you intended.

## Sources

This guide combines [the anonymous branches lesson](https://steveklabnik.github.io/jujutsu-tutorial/branching-merging-and-conflicts/anonymous-branches.html) and [the merging and rebasing lesson](https://steveklabnik.github.io/jujutsu-tutorial/branching-merging-and-conflicts/merging.html).