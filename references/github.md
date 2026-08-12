# Working with GitHub and GitLab

A local Jujutsu stack does not need a name. A Git forge does. Treat bookmarks as publication pointers that you create or move when work crosses the remote boundary.

## Publish a stack

Create commits first:

```bash
jj new main
# edit
jj commit -m "Refactor parser"
# edit
jj commit -m "Add parser option"
```

After `jj commit`, `@` is normally the new empty working-copy commit, so the completed stack ends at `@-`.

For a one-off generated bookmark:

```bash
jj git push -c @-
```

For a durable review name:

```bash
jj bookmark create parser-option -r @-
jj bookmark track parser-option
jj git push -b parser-option
```

Bookmarks do not advance automatically. After adding another commit, move the bookmark before pushing:

```bash
jj bookmark move parser-option --to @-
jj git push -b parser-option
```

Agents must not push unless the user explicitly requests it. Use `jj git push --dry-run` to inspect the proposed update when needed.

## Update local work

Jujutsu separates network transfer from history rewriting:

```bash
jj git fetch
jj rebase -b @ -o main
```

Fetch updates remote state. Rebase moves the selected local branch onto the updated main line. If several anonymous branches are outstanding, rebase each branch or pass several `-b` selectors.

## Address review comments

If the project keeps follow-up commits:

```bash
jj new parser-option
# edit and review
jj commit -m "Address review comments"
jj bookmark move parser-option --to @-
jj git push -b parser-option
```

If the project expects clean rewritten commits:

```bash
jj new parser-option-
# edit and review
jj squash -u
jj git push -b parser-option
```

The trailing `-` selects the bookmark's parent. Jujutsu pushes rewritten bookmark history with force-with-lease style safety, so fetch if the remote has moved rather than bypassing the check.

## Work with another contributor's branch

A fetched remote bookmark does not have to become a local bookmark. Build on it directly:

```bash
jj new feature@origin
```

Configure `remotes.origin.auto-track-bookmarks = "*"` only if you really want every remote bookmark tracked locally.

## Use a fork and upstream remote

```bash
jj git clone --remote upstream https://github.com/upstream-org/repo
jj git remote add origin git@github.com:your-org/your-fork
```

Set the defaults in Jujutsu config:

```toml
[git]
fetch = "upstream"
push = "origin"
```

Use a list for `git.fetch` when you also synchronize your fork between machines.

## Use GitHub CLI

In a colocated repository, `gh` sees `.git/` normally. In a non-colocated repository, point it at Jujutsu's backing Git repository:

```bash
GIT_DIR=$(jj git root) gh issue list
```

A direnv `.envrc` can export that value for repeated use.

GitLab-specific push options can be forwarded with repeated `-o` flags, but they are server features rather than Jujutsu workflow. Use them only when the hosting project already relies on them.

## Source

This guide combines the common flows from the official [GitHub and GitLab guide](https://docs.jj-vcs.dev/latest/github/).