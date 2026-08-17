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

### The Squash Workflow (Recommended)

The workflow preferred by jj's creator, and the default way to work: treat `@` as
a staging area and pull changes into a described change with `jj squash`.
**Always create your commit message before writing code.**

First decide where the change belongs:

- **New change** — a distinct piece of work: start from an empty, undescribed `@`
  and describe it before coding: `jj desc -m "print goodbye as well as hello"`.
  If `@` already has changes or a description, do not blindly run `jj new`: inspect
  `jj --no-pager show @`, then finish or split that work, or start from the intended
  base with `jj new <base>`.
- **Existing change** — fixing or extending a commit you already made (e.g. review
  feedback): update its description with `jj describe <change-id> -m "..."`, then
  squash the new work into it (`jj squash --into <change-id>` below).

Then the loop:

1. Create an empty scratch change on top: `jj new` — this `@` acts like the git index
2. Do the work in `@`
3. Move it into the target change: `jj squash` amends the parent commit; `jj squash --into <change-id>` targets any commit — not just the one before `@`
   - `jj squash <paths>` moves only the given paths
   - `jj squash -i` opens an interactive TUI — not agent-safe, avoid
   - `jj abandon` on `@` discards scratch changes you don't want

When **finishing** work, use exactly one of these paths — new edits must never
fold into a finished, described commit:

- In the squash/describe-first workflow, run `jj new` once after the current `@` is complete: the finished change becomes `@-` and the new `@` is empty.
- After `jj commit -m "message" [<paths>]`, do not run `jj new` again: `jj commit` already created a new `@`. Inspect it with `jj st`; if it contains unselected changes, keep them in `@` and describe, split, or finish them before starting unrelated work.

```bash
# First, describe what you intend to do
jj desc -m "Add user authentication to login endpoint"

# Then make your changes - they automatically become part of this commit
# ... edit files ...

# Check status
jj st
```

### Alternative: The Edit Workflow

Work directly on feature changes instead of squashing; `jj new -B @` inserts a new change *before* the current one with automatic descendant rebase. Full steps: [edit-workflow.md](references/edit-workflow.md).

### Creating Atomic Commits

Each commit should represent ONE logical change, with a Conventional Commits message: `type(scope): description` — e.g. `feat: add login endpoint`, `fix(user-auth): handle null pointer`, `docs: update README`, `refactor: remove deprecated endpoints`. Imperative, lowercase, no final period. Common types: `feat`, `fix`, `docs`, `refactor`, `chore`, `test`.

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
jj new -m "Commit message"        # or with a message
jj edit <change-id>               # edit an existing commit (@ becomes it)
jj prev -e                        # edit the previous commit
jj next -e                        # edit the next commit
```
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

Inspect the operation log before reversing anything — another process or workspace may have created the latest operation:

```bash
jj --no-pager op log
jj undo                # only when the latest operation is the one you intend to reverse
jj op revert <id>      # reverse an older operation while preserving later work
jj st
```

Use `jj undo` only for the verified latest operation; reserve `jj op restore` for restoring the entire repository view. See [operation-log.md](references/operation-log.md) for the recovery sequence.

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

A workspace is a working copy plus its associated repo — jj's equivalent of `git worktree`. One repo can have several, each with its own `@` (labeled `<workspace-name>@` in `jj log`), sharing commits and bookmarks. Useful for running a long build/test in one workspace while editing in another. See [workspaces.md](references/workspaces.md) for commands, isolation/stale-working-copy semantics, and agent guidance.

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

Only for colocated repos (both `.jj/` and `.git/`): finish the jj change and verify both `jj st` and `git status --short` are clean before any git checkout, commit, merge, or rebase. Full flow and the colocation decision: [colocated-repos.md](references/colocated-repos.md).

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

Conflicts are stored in commits — locate them first instead of assuming `@`:

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
jj --no-pager log -r 'conflicts()'   # should no longer list the revision
```

Do not use `jj resolve` in an agent — it launches an interactive merge tool. See [conflicts.md](references/conflicts.md) for marker formats, divergence, and bookmark conflicts.

## Preserving Commit Quality

**IMPORTANT**: Because commits are mutable, always refine them before considering work done:

1. **Review your commit**: `jj --no-pager show @` or `jj --no-pager diff --git`
2. **Is it atomic?** One logical change per commit
3. **Is the message clear?** Use Conventional Commits (`type: description`, lowercase imperative, no final stop): e.g. `feat: add login endpoint`, `fix: handle null pointer in payment processor`, `chore: update dependencies`
4. **Are there unrelated changes?** Use `jj commit -m "<message>" <paths>` to finish the related files; inspect the remaining changes left in the new `@`
5. **Should changes be elsewhere?** Use `jj squash` or `jj absorb`
6. **Verify the whole stack**: run formatters/linters/tests across every commit with `jj run -r '::@'` — see [run.md](references/run.md)

## Running Commands Across Revisions (`jj run`)

`jj run` applies one command to many revisions: it checks out each selected revision in an isolated working copy, runs the command, and amends the revision with the result (descendants rebased automatically). Typical uses: format, lint, or test every commit in a stack; run a pre-commit hook; regenerate files. Runs never touch your own working copy.

- `jj run -r <revset> -- <cmd>` — select revisions with `-r` (default `@`), parallelize with `-j <n>`; use `--` before the command so its flags aren't parsed by jj
- `--ignore-changes` — read-only checks (tests, linters) without rewriting any commit; also permits immutable commits
- `--ignore-errors` — continue past failing commands; `--clean` starts each commit from a fresh checkout; `--passthrough` for TTY output (single job only)

Full options, examples, and rewrite semantics: [run.md](references/run.md).
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
3. [`references/colocated-repos.md`](references/colocated-repos.md): decide whether jj and Git should share a working copy, and learn how to mix tools and switch branches safely.
4. [`references/git-command-table.md`](references/git-command-table.md): translate familiar Git intentions into Jujutsu's working-copy, revision, bookmark, and operation model.
5. [`references/faq.md`](references/faq.md): diagnose missing commits, stationary bookmarks, partial changes, empty merges, and other common surprises.
6. [`references/github.md`](references/github.md): publish stacks, update reviews, work with forks, and use generated or named bookmarks on GitHub and GitLab.
7. [`references/windows.md`](references/windows.md): set line endings, quote PowerShell revsets, avoid WSL permission noise, and configure symlinks.
8. [`references/operation-log.md`](references/operation-log.md): inspect earlier repository views and choose among undo, revert, restore, and per-change evolution.
9. [`references/installation.md`](references/installation.md): install jj, verify its Git requirement, set author identity, and create a first repository.
10. [`references/workspaces.md`](references/workspaces.md): add a second working copy for parallel builds/tests, and handle staleness and divergence.
11. [`references/run.md`](references/run.md): apply a formatter, linter, or test across many revisions with `jj run`.
12. [`references/edit-workflow.md`](references/edit-workflow.md): work directly on feature changes, and insert changes before the current commit with `jj new -B`.

For revision selection syntax used throughout these guides, read [`references/revsets.md`](references/revsets.md).
## Source

Incorporates [danverbraganza/jujutsu-skill](https://github.com/danverbraganza/jujutsu-skill), licensed under MIT; see [`LICENSE`](LICENSE).
