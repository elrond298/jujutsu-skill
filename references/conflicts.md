# Conflicts and divergent changes

Use this guide when `jj st` or `jj log` reports a conflict, a change ID is marked divergent, or a bookmark has `??` after its name. These are related recovery problems, but they are not the same state.

## File conflicts are committed states

Jujutsu records a conflict inside a commit. A merge or rebase can therefore finish even when file contents cannot be combined automatically. You can keep working elsewhere, move the conflicted commit, or resolve it later. There is no interrupted operation and no `rebase --continue`.

Find the affected commits before editing:

```bash
jj st
jj --no-pager log -r 'conflicts()'
jj --no-pager show CONFLICTED_REV
```

## Resolve with a reviewable child commit

The safest workflow keeps the resolution separate until you have inspected it:

```bash
jj new CONFLICTED_REV
# edit each conflicted file and remove all conflict markers
jj --no-pager diff --git
jj squash -u
jj st
jj --no-pager log -r 'conflicts()'
```

`jj new` materializes the conflict in the working copy. After you replace the marked region with the intended final text, `jj squash -u` moves that resolution into the conflicted parent while keeping its description. Descendants are automatically rebased, so one resolution can clear conflicts further up the stack.

A human can run `jj resolve` to launch a configured merge tool. Agents should edit the files directly because `jj resolve` is interactive.

For a tiny conflict, `jj edit CONFLICTED_REV` followed by direct editing also works. The child-and-squash flow is easier to inspect and undo.

## Read the default markers

The outer markers bound one conflict:

```text
<<<<<<< conflict 1 of 1
%%%%%%% diff from: BASE
        to: SIDE_A
 old
-value
+new value
+++++++ SIDE_B
OTHER SNAPSHOT
>>>>>>> conflict 1 of 1 ends
```

`+++++++` begins a complete snapshot. `%%%%%%%` begins a diff that should be applied to a snapshot. Resolve the example by choosing the desired snapshot and applying the relevant diff, then delete every marker line.

If the default form is hard to read, choose another display style:

```bash
jj config set --repo ui.conflict-marker-style snapshot
jj config set --repo ui.conflict-marker-style git
```

The Git style supports only two sides. Snapshot style also works for merges with more than two parents.

## Divergence is an identity problem

A divergent change means that multiple visible commits share one change ID. It often happens when two workspaces or processes rewrite the same change, or when an old hidden version becomes visible again.

The bare change ID is ambiguous. Refer to each version by commit ID or by its change offset, such as `changeid/0` and `changeid/1`.

Inspect both versions first:

```bash
jj --no-pager log -r 'CHANGE_ID/0 | CHANGE_ID/1'
jj --no-pager show COMMIT_ID
```

Then choose one outcome:

```bash
# One version is obsolete
jj abandon UNWANTED_COMMIT_ID

# Both should remain as separate changes
jj metaedit --update-change-id COMMIT_ID

# Both contents belong in one change
jj squash --from SOURCE_COMMIT_ID --into TARGET_COMMIT_ID -u
```

Leaving immutable divergent commits alone is valid. The cost is that the shared change ID stays ambiguous.

## A conflicted bookmark is a pointer problem

A bookmark with `??` points to more than one commit. List its targets, choose the intended commit ID, and move the bookmark explicitly:

```bash
jj --no-pager bookmark list
jj bookmark move NAME --to COMMIT_ID
```

Do not treat this as a file conflict or solve it by editing files.

## Sources

This guide combines [Steve Klabnik's conflict lesson](https://steveklabnik.github.io/jujutsu-tutorial/branching-merging-and-conflicts/conflicts.html), the official [first-class conflicts guide](https://docs.jj-vcs.dev/latest/conflicts/), and the official [divergent changes guide](https://docs.jj-vcs.dev/latest/guides/divergence/).