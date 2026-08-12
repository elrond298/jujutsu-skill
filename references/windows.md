# Working on Windows

Jujutsu's graph and commit model are platform independent. Windows users mainly need to settle line endings, shell quoting, filesystem location, and symlink behavior before those details create noisy commits.

## Choose one line-ending policy

Jujutsu uses `working-copy.eol-conversion`. It does not read `.gitattributes` or Git's `core.autocrlf`, so colocated repositories can disagree about the same file.

Check the project policy first, then keep the Git and Jujutsu settings compatible. To keep Jujutsu from converting line endings:

```powershell
PS> jj config set --repo working-copy.eol-conversion none
```

Configure Git separately for the same checkout policy. For example, `core.autocrlf input` keeps LF on checkout while normalizing CRLF on commit.

If a new colocated workspace is dirty only because of line endings:

1. Confirm with `jj diff` that there are no real content edits.
2. Correct both settings.
3. Run `jj abandon` to recreate the working copy from committed content.

`jj abandon` discards the current working-copy change, so do not use it before checking the diff.

## Quote revsets in PowerShell

PowerShell gives `@` its own meaning. Quote or escape it:

```powershell
PS> jj log -r '@'
PS> jj log -r `@
```

A revset alias avoids repeated quoting:

```powershell
PS> jj config set --user revset-aliases.HEAD '@'
PS> jj log -r HEAD
```

## Pager choices

Windows uses Jujutsu's integrated `streampager` unless `ui.pager` is configured. To use Git for Windows' `less`:

```powershell
PS> jj config set --user ui.pager '["C:\\Program Files\\Git\\usr\\bin\\less.exe", "-FRX"]'
PS> jj config set --user ui.paginate auto
```

Agents should still pass `--no-pager` on output commands.

## Keep WSL repositories off Windows mounts

Files under `/mnt/c` and similar mounts can appear executable to WSL. Since Jujutsu snapshots the working copy automatically, this can record execute-bit changes across the repository.

If only WSL needs the repository, clone it under the Linux filesystem, such as `~/my-repo`. If both Windows and WSL need access, use a separate Jujutsu workspace in the Linux filesystem and open that workspace only from WSL.

## Enable symlinks deliberately

Windows needs Developer Mode and Windows 10 version 14972 or newer to materialize symlinks as symlinks. Otherwise Jujutsu writes ordinary files. A colocated Git repository also needs:

```powershell
PS> git config core.symlinks true
```

Set this before checking out a repository that depends on symlinks.

## Source

The platform details come from the official [Windows guide](https://docs.jj-vcs.dev/latest/windows/).