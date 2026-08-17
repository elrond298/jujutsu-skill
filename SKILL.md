---
name: jujutsu
description: jujutsu (jj, jj-vcs) workflow description, and command usage for subcommands like merge, rebase, split, bookmark, git integration, etc. MUST be loaded first if the repo is managed by jj (containing .jj)
license: MIT
---

# Jujutsu (jj) Version Control System

This skill helps you work with Jujutsu, a Git-compatible VCS with mutable commits and automatic rebasing.

**Reviewed with jj v0.44.0** - Commands may differ in other versions.

## Important: Automated/Agent Environment

When running as an agent:

1. **Always use `--no-pager`** to prevent commands from opening an interactive pager (like `less`), which will hang the agent:

```bash
# Always use --no-pager on commands that produce output
jj --no-pager log          # NOT: jj log
jj --no-pager diff         # NOT: jj diff
jj --no-pager show <id>    # NOT: jj show <id>
```

2. **Pass descriptions inline** instead of relying on editor prompts. For `squash`, either keep the destination description with `-u` or provide the final description with `-m`:

```bash
jj desc -m "message"          # NOT: jj desc
jj squash -u                  # Keep destination description
jj squash --into <rev> -u     # Squash elsewhere without an editor
jj squash -m "final message"  # Explicitly set the resulting description
```

Editor-based commands will fail in non-interactive environments.

3. **Verify operations with `jj st`** after mutations (`squash`, `abandon`, `rebase`, `restore`) to confirm the operation succeeded.

## Core Concepts

### The Working Copy is a Commit

In jj, your working directory is always a commit (referenced as `@`). Changes are automatically snapshotted when you run any jj command. There is no staging area.

The working copy already records your changes, but `jj commit -m "message"` is still a useful way to finish the whole change: it describes `@` and creates a new working-copy commit on top. With filesets, `jj commit -m "message" <paths>` keeps the selected changes in the finished commit and moves everything else into the new `@` (see "Splitting Commits").

### Commits Are Mutable

Unlike Git, jj lets you rewrite mutable local commits with operations such as `describe`, `squash`, `rebase`, and `absorb`. Revisions selected by `immutable_heads()` — normally trunk and other public history — are protected from rewriting; build new work on top instead of bypassing that protection. See "Essential Workflow" below for the recommended working pattern.

### Change IDs vs Commit IDs

- **Change ID**: A stable identifier (like `tqpwlqmp`) that persists when a commit is rewritten — prefer these when referencing commits
- **Commit ID**: A content hash (like `3ccf7581`) that changes when commit content changes

### Revsets

jj uses a revset language to select commits in commands. Common revsets:

- `@` — the working copy commit
- `@-` — the parent of the working copy
- `::@` — all ancestors of `@`
- `@::` — all descendants of `@`
- `trunk()..@` — commits between trunk and `@` (your branch)
- `bookmarks()` — all commits with bookmarks

Use revsets with `-r` flags: `jj log -r 'trunk()..@'`

## Essential Workflow

### Starting Work: Describe First, Then Code

**Always create your commit message before writing code:**

Run `jj st` and start only from an empty, undescribed `@`. If `@` already has changes or a description, do not blindly run `jj new`: inspect `jj --no-pager show @`, then finish or split that work, or create the new change from the intended base with `jj new <base>`.

When **finishing** work, use exactly one of these paths:

- In the describe-first workflow, run `jj new` once after the current `@` is complete. The finished change becomes `@-` and the new `@` is empty.
- After `jj commit -m "message" [<paths>]`, do not run `jj new` again. `jj commit` already created a new `@`; inspect it with `jj st`. If it contains unselected changes, keep them in `@` and describe, split, or finish them before starting unrelated work.

New edits must never fold into a finished, described commit.

```bash
# First, describe what you intend to do
jj desc -m "Add user authentication to login endpoint"

# Then make your changes - they automatically become part of this commit
# ... edit files ...

# Check status
jj st
```

### Creating Atomic Commits

Each commit should represent ONE logical change. Use this format for commit messages:

```
Examples:
- "Add validation to user input forms"
- "Fix null pointer in payment processor"
- "Remove deprecated API endpoints"
- "Update dependencies to latest versions"
```

### Viewing History

```bash
# View recent commits
jj --no-pager log

# View with patches
jj --no-pager log -p

# View specific commit
jj --no-pager show <change-id>

# View diff of working copy (use --git for familiar +/- format)
jj --no-pager diff --git

# View diff of a specific change — revisions need -r; positional args are paths
jj --no-pager diff --git -r <change-id>
```

**IMPORTANT: `jj diff` output format**: The default `jj diff` output uses a side-by-side line number format (e.g. `26   26:`) that looks very different from git's `+`/`-` prefix format. This is **normal and correct** — it is NOT corrupted or showing stale content. However, to avoid confusion, **always use `jj diff --git`** to get standard unified diff format with `+`/`-` lines.

### Moving Between Commits

```bash
# Create a new empty commit on top of current
jj new

# Create new commit with message
jj new -m "Commit message"

# Edit an existing commit (working copy becomes that commit)
jj edit <change-id>

# Edit the previous commit
jj prev -e

# Edit the next commit
jj next -e
```

## Workflows

Two established workflows from [Steve Klabnik's jujutsu tutorial](https://steveklabnik.github.io/jujutsu-tutorial/real-world-workflows/intro.html).

### The Squash Workflow

The workflow preferred by jj's creator: treat `@` as a staging area and pull
changes into a described change with `jj squash`.

1. Describe the work on the current (empty) change: `jj desc -m "print goodbye as well as hello"`
2. Create an empty scratch change on top: `jj new` — this `@` acts like the git index
3. Do the work in `@`, then move it into the described change: `jj squash`
   - `jj squash <paths>` moves only the given paths
   - `jj squash -i` opens an interactive TUI — not agent-safe, avoid
   - `jj abandon` on `@` discards scratch changes you don't want
4. `jj squash` works on any change and its parent, not just the working copy

### The Edit Workflow

Work directly on feature changes, and insert new changes *before* the current
one when they must land first.

1. Start a feature change: `jj new -m "only print hello world"`
2. Do the work in it; if that's all you need, you're done
3. To add a change that must come BEFORE the current one:
   `jj new -B @ -m "add more comments"` — inserts before `@` and automatically
   rebases descendant commits (always succeeds; conflicts resolve later)
4. Make that change, then return to the main change with `jj edit <change-id>`
   or `jj next --edit` (edits the child of `@`)

Note: `jj new -B` rebases descendants — change IDs stay stable while commit
IDs change.

## Refining Commits
### Squashing Changes

Move changes from the current commit into its parent or another revision:

```bash
# Keep the destination description and avoid an editor
jj squash -u

# Squash into a specific revision
jj squash --into <change-id> -u
```

Without `-u` or `-m`, `squash` opens an editor when both source and destination have descriptions. `jj squash -i` opens an interactive diff UI. Avoid both interactive modes in agent environments.

### Splitting Commits

Two operations split changes; pick by what is being split.

**Working copy — `jj commit <paths>`**: commit only some files, keep the rest
in the working copy:

```bash
jj commit -m "type(scope): description" path/to/file1 path/to/file2
```

The given paths stay in the described commit; all other changes move to a
new working-copy commit. This is the simplest split and the right tool when
the changes are still uncommitted in `@`.

**Existing revision — `jj split`**: cut one commit into two. Bare `jj split`
(and `-i`, `--tool`) launches a diff editor and hangs in agent environments —
avoid them. The non-interactive form needs no editor at all:

```bash
jj split -m "message for the first commit" path/to/fileA path/to/fileB
```

Filesets select the changes that remain in the original commit, in place;
everything else moves into a new child commit stacked on top. `-m` sets the
description of the original commit without opening a description editor; the
new child keeps the original description, if any. Without `-m`, a
description editor still opens, so always pass `-m` in agents. Use `-r <rev>`
to split a revision other than `@`; splitting an empty commit is unsupported
(that is what `jj new` is for). Advanced variants (`--parallel`, `--onto`,
`--insert-before/after`) move the two parts elsewhere — see `jj split --help`.

**Warning**: `jj restore <paths>` discards changes to those paths — it does
not move them into another commit. Use `jj commit <paths>` when you want to
keep the changes.

### Absorbing Changes

Automatically distribute changes to the commits that last modified those lines:

```bash
# Absorb working copy changes into appropriate ancestor commits
jj absorb
```

### Abandoning Commits

Remove a commit entirely (descendants are rebased to its parent):

```bash
jj abandon <change-id>
```

### Undoing Operations

Inspect the operation log before reversing anything; another process or workspace may have created the latest operation:

```bash
jj --no-pager op log

# Use only when the latest operation is the one you intend to reverse
jj undo
jj st

# Reverse an older operation while preserving later work
jj op revert <operation-id>
jj st
```

Use `jj undo` only for the verified latest operation. For older mistakes, use targeted `jj op revert`; reserve `jj op restore` for restoring the entire repository view. See [operation-log.md](references/operation-log.md) for the recovery sequence.

### Rebasing Commits

Move commits to a different parent:

```bash
# Rebase current branch onto a destination
jj rebase -b @ -o <destination>

# Rebase a specific revision (without descendants) onto a destination
jj rebase -r <change-id> -o <destination>

# Rebase a revision and all its descendants
jj rebase -s <change-id> -o <destination>

# Rebase onto trunk (common: update your branch to latest main)
jj rebase -b @ -o main
```

### Restoring Files

Discard changes to specific files or restore files from another revision:

```bash
# Discard all uncommitted changes in working copy (restore from parent)
jj restore

# Discard changes to specific files
jj restore path/to/file.txt

# Restore files from a specific revision
jj restore --from <change-id> path/to/file.txt
```

## Working with Bookmarks (Branches)

Bookmarks are jj's equivalent to git branches:

```bash
# Create a bookmark at current commit
jj bookmark create my-feature -r@

# Move bookmark to a different commit
jj bookmark move my-feature --to <change-id>

# List bookmarks
jj --no-pager bookmark list

# Delete a bookmark
jj bookmark delete my-feature
```

## Workspaces

A **workspace** is a working copy plus its associated repo. One repo can have multiple workspaces — each with its own working directory and working-copy commit (`@`) — all sharing the same commits, operations, and bookmarks. This is jj's equivalent of `git worktree`.

Useful for running a long build or test in one workspace while editing in another. Workspaces are a rarely-needed feature; consult the [official docs](https://docs.jj-vcs.dev/latest/working-copy/#workspaces) for anything beyond the basics below.

### Common commands

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

### Key semantics

- **Isolation by default.** `jj workspace add` gives the new workspace its own fresh empty commit; workspaces don't start out sharing `@`, and on-disk files are never live-mirrored between them.
- **Propagation at command boundaries.** Each jj command snapshots the current workspace's files and reads the op log, so it sees commits/bookmarks made by other workspaces. There is no filesystem watcher.
- **Stale working copy.** If another workspace rewrites this workspace's `@` (e.g. via `jj squash`, `rebase`, `abandon`), jj refuses commands here until you run `jj workspace update-stale`. Same recovery path if a command was interrupted mid-update.
- **Shared `@` is sharp-edged.** `jj edit <id>` lets two workspaces point at the same change without warning. When one mutates it, the other goes stale; if the stale one had un-snapshotted edits, `update-stale` preserves them as a **divergent commit** (same change ID, shown as `xyz??` in `jj log`) that you must resolve. Avoid sharing `@` unless both workspaces are read-only.

### Agent guidance

- Always pass `--no-pager` to `jj workspace list`.
- Don't `jj edit` a change another workspace already has as its `@` — main cause of accidental divergence.
- Don't `rm -rf` a workspace directory without also running `jj workspace forget <name>`.

## Git Integration

### Working with Existing Git Repos

```bash
# Clone a git repository
jj git clone <url>

# Initialize jj in an existing git repo (colocation is the default)
jj git init
```

### Fetching Remote Changes

```bash
# Fetch all branches from the default remote
jj git fetch

# Fetch from a specific remote
jj git fetch --remote <remote-name>

# Fetch specific branches
jj git fetch -b <branch-name>
```

After fetching, rebase your work onto the updated trunk: `jj rebase -o main`

### Switching Between jj and git (Colocated Repos Only)

**This section only applies to colocated repos** (where both `.jj/` and `.git/` exist). In a non-colocated workspace, use `jj git` commands; the backing Git repository is hidden inside `.jj/`.

In a colocated repository, you can use both jj and git commands with care:

Git can switch branches only when the shared working tree is clean from Git's perspective. A bare `jj st` merely snapshots changes; it does not make Git clean. Finish the jj change and create an empty `@` first:

```bash
# Finish the current jj change; this creates a new empty @
jj commit -m "message"
jj st
git status --short

# Continue only when both commands report no working-copy changes
git checkout <branch-name>
```

After Git work, run `jj st`. That command imports Git's HEAD and refs and resets the jj working-copy parent when needed; do not run `jj edit` merely to switch back. Choose `jj edit <change-id>` only when you intentionally want to rewrite that existing change.

Prefer jj for mutations and Git for read-only tools. Before any Git checkout, commit, merge, or rebase, ensure both `jj st` and `git status --short` are clean. See [colocated-repos.md](references/colocated-repos.md) for the full workflow.

### Pushing Changes

When the user asks you to push changes:

```bash
# Push a specific bookmark to the remote
jj git push -b <bookmark-name>

# Example: push the main bookmark
jj git push -b main
```

**Before pushing, ensure:**
1. Your bookmark points to the correct commit (bookmarks don't auto-advance like git branches)
2. The commits are refined and atomic
3. The user has explicitly requested the push

**IMPORTANT**: Unlike git branches, jj bookmarks do not automatically move when you create new commits. You must manually update them before pushing:

```bash
# Move an existing bookmark to the current commit
jj bookmark move my-feature --to @

# Then push it
jj git push -b my-feature
```

If no bookmark exists for your changes, create one first:

```bash
# Create a bookmark at the current commit
jj bookmark create my-feature

# Then push it
jj git push -b my-feature
```

## Handling Conflicts

jj stores conflicts in commits, so first locate the conflicted revision instead of assuming it is `@`:

```bash
jj st
jj --no-pager log -r 'conflicts()'
jj --no-pager show <conflicted-revision>

# Resolve in a reviewable child
jj new <conflicted-revision>
# edit conflicted files and remove all conflict markers
jj --no-pager diff --git
jj squash -u
jj st
jj --no-pager log -r 'conflicts()'
```

Do not use `jj resolve` in an agent because it launches an interactive merge tool. The final `conflicts()` query should no longer list the resolved revision. See [conflicts.md](references/conflicts.md) for marker formats, divergence, and bookmark conflicts.

## Preserving Commit Quality

**IMPORTANT**: Because commits are mutable, always refine them before considering work done:

1. **Review your commit**: `jj --no-pager show @` or `jj --no-pager diff --git`
2. **Is it atomic?** One logical change per commit
3. **Is the message clear?** Use imperative verb phrase in sentence case format with no full stop: e.g. "Add login endpoint", "Fix null pointer in payment processor", "Remove deprecated API endpoints"
4. **Are there unrelated changes?** Use `jj commit -m "<message>" <paths>` to finish the related files; inspect the remaining changes left in the new `@`
5. **Should changes be elsewhere?** Use `jj squash` or `jj absorb`
6. **Verify the whole stack**: run formatters/linters/tests across every commit with `jj run -r '::@'` (below)

## Running Commands Across Revisions (`jj run`)

`jj run` checks out each selected revision in an isolated working copy, runs a
command, and amends the revision with the resulting working copy — effectively
"apply this command to every commit", with automatic descendant rebasing.

The subprocesses run in temporary working copies (under `.jj/`), so your own
working copy and files are never touched — you can keep editing while `jj run`
runs. Like all rewrite commands, it refuses immutable/public commits; only
`--ignore-changes` (no rewrite at all) or `--ignore-immutable` bypass that.

Typical uses: apply a formatter or pre-commit hook across a stack, regenerate
files, or test each commit in isolation.

- Select revisions with `-r <revset>` (defaults to `@`). Parallelize with `-j <n>`
  (also configurable via the `run.jobs` config setting).
- The command runs from the directory `jj run` was invoked from; use `--root` to
  run from each commit's working-copy root instead.
- The command sees `JJ_CHANGE_ID`, `JJ_COMMIT_ID`, and `JJ_WORKSPACE_ROOT`.
- Use a `--` separator so command arguments starting with `-` are not parsed as
  jj flags: `jj run -r '::@' -- cargo fmt`
- By default, working copies are reused between invocations so build artifacts
  survive; `--clean` starts each commit from a fresh checkout.
- `--ignore-changes` runs the command but discards working-copy changes and
  rewrites nothing — ideal for read-only checks (tests, linters), and it works on
  immutable commits without `--ignore-immutable`.
- `--ignore-errors` continues past failing commands; a failed command's changes
  are not saved, successful ones are applied atomically.
- `--passthrough` connects stdout/stderr to the terminal (TTY behavior, colors,
  progress bars) but allows only one job.

```bash
# Run pre-commit on your local work
jj run -j 4 -- pre-commit run .github/pre-commit.yaml

# Apply a formatter to every commit in the current stack
jj run -r '::@' -- cargo fmt

# Lint all descendants without rewriting anything
jj run -r '::@' --ignore-changes -- cargo clippy

# Build across a stack, reusing the target dir between runs
jj run -r '::@' -- bazel build //some/target:somewhere
```

Warning: except with `--ignore-changes`, `jj run` rewrites the selected commits,
so commit IDs change. Descendants are rebased to propagate the diff by default;
`--restore-descendants` keeps descendants' content unchanged instead.

## Quick Reference


| Action | Command |
|--------|---------|
| Describe commit | `jj desc -m "message"` |
| View status | `jj st` |
| View log | `jj --no-pager log` |
| View diff | `jj --no-pager diff --git` |
| New commit | `jj new -m "message"` (only after inspecting and finishing existing `@`) |
| Edit commit | `jj edit <id>` |
| Squash to parent | `jj squash -u` |
| Auto-distribute | `jj absorb` |
| Rebase current branch | `jj rebase -b @ -o <destination>` |
| Abandon commit | `jj abandon <id>` |
| Undo latest verified operation | inspect `jj --no-pager op log`, then `jj undo` |
| Restore files | `jj restore [paths]` |
| Create bookmark | `jj bookmark create <name>` |
| Fetch remote | `jj git fetch` |
| Push bookmark | `jj git push -b <name>` |
| Add workspace | `jj workspace add <path>` |
| List workspaces | `jj --no-pager workspace list` |
| Forget workspace | `jj workspace forget [name]` |
| Fix stale working copy | `jj workspace update-stale` |
| Run command across revisions | `jj run -r <revset> -- <cmd>` |

## Best Practices Summary

1. **Describe first**: Set the commit message before coding
2. **One change per commit**: Keep commits atomic and focused
3. **Use change IDs**: They're stable across rewrites
4. **Refine commits**: Leverage mutability for clean history
5. **Embrace the workflow**: No staging area, no stashing - just commits

## Learning guides

Read the focused guide that matches the task. Each guide teaches a workflow and links its source material at the end.

1. [`references/anonymous-branches-and-merging.md`](references/anonymous-branches-and-merging.md): learn how unnamed forks work, then choose between a multi-parent merge and a rebase.
2. [`references/conflicts.md`](references/conflicts.md): resolve committed file conflicts, divergent change IDs, and conflicted bookmarks without confusing the three states.
3. [`references/colocated-repos.md`](references/colocated-repos.md): decide whether jj and Git should share a working copy and learn how to mix tools safely.
4. [`references/git-command-table.md`](references/git-command-table.md): translate familiar Git intentions into Jujutsu's working-copy, revision, bookmark, and operation model.
5. [`references/faq.md`](references/faq.md): diagnose missing commits, stationary bookmarks, partial changes, empty merges, and other common surprises.
6. [`references/github.md`](references/github.md): publish stacks, update reviews, work with forks, and use generated or named bookmarks on GitHub and GitLab.
7. [`references/windows.md`](references/windows.md): set line endings, quote PowerShell revsets, avoid WSL permission noise, and configure symlinks.
8. [`references/operation-log.md`](references/operation-log.md): inspect earlier repository views and choose among undo, revert, restore, and per-change evolution.
9. [`references/installation.md`](references/installation.md): install jj, verify its Git requirement, set author identity, and create a first repository.

For revision selection syntax used throughout these guides, read [`references/revsets.md`](references/revsets.md).
## Source

Incorporates [danverbraganza/jujutsu-skill](https://github.com/danverbraganza/jujutsu-skill), licensed under MIT; see [`LICENSE`](LICENSE).
