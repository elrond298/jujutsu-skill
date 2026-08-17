# Colocated Jujutsu and Git repositories

Use this guide when a working directory contains both `.jj/` and `.git/`, or when Git-based tools must work beside Jujutsu.

## What colocation does

A colocated workspace gives Jujutsu and Git one working copy. Each `jj` command imports Git refs before it runs and exports Jujutsu state afterward. This makes IDEs, build tools, and read-only Git commands work without a separate checkout.

Colocation is the default for current Git-backed clones and repositories:

```bash
jj git clone URL
jj git init
```

Use `--no-colocate` when you want the backing Git repository hidden inside `.jj/` and do not need ordinary Git tools in the working directory.

## Use one tool as the writer

The easiest rule is to use `jj` for mutations and Git only for inspection:

```bash
jj st
git log --oneline
```

Mixing mutating commands is supported, but it is easier to create bookmark conflicts or divergent changes. Jujutsu has no current branch, so Git usually sees a detached HEAD after a `jj` command. If you must make a Git mutation, first inspect the working copy, then tell Git which branch to use:

```bash
jj st
git switch BRANCH
# run the required Git command
jj st
jj --no-pager op log
```

The next `jj` command imports the Git result. The operation log records that import, so `jj undo` or `jj op restore` can recover local repository state if the result is wrong.

## Switch branches cleanly

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

Before any Git checkout, commit, merge, or rebase, ensure both `jj st` and `git status --short` are clean.
## Know what does not translate

Colocation does not make every Git feature part of Jujutsu:

- Jujutsu ignores Git's staging area and unfinished rebase or merge state.
- Git tools cannot interpret Jujutsu's stored representation of conflicted commits correctly.
- `.gitignore`, Git remotes, authentication, merge commits, lightweight tags, and signed commits work.
- `.gitattributes`, Git hooks, submodules, partial clones, Git LFS, and `git worktree` are not supported as Jujutsu features.
- Use `jj workspace` instead of `git worktree`, and `jj sparse` instead of Git sparse checkout.

If a Git tool shows `.jjconflict-*` directories, stop using Git on that commit. Inspect it with `jj st` and return to a sound Jujutsu state before continuing.

## Decide whether to colocate

Start colocated when you are learning or depend on Git-aware tools. Prefer non-colocated when all mutations happen through Jujutsu and you have a concrete problem with automatic Git import, such as a repository with very many refs or background IDE Git operations.

Check or change the mode with:

```bash
jj git colocation status
jj git colocation enable
jj git colocation disable
```

Do not convert manually. The built-in commands update repository paths and Git HEAD correctly, including on Windows.

## Source

This guide teaches the workflow described by the official [Git compatibility and colocation documentation](https://docs.jj-vcs.dev/latest/git-compatibility/#colocated-jujutsugit-repos).