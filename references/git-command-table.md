# Git to Jujutsu command guide

Translate the goal of a Git command, not its spelling. Jujutsu has no staging area, its working copy is a commit, and most commands can target any revision.

## Set up and synchronize

| Goal | Git | Jujutsu |
| --- | --- | --- |
| Create a repository | `git init` | `jj git init [--no-colocate]` |
| Clone | `git clone URL [DIR]` | `jj git clone URL [DIR]` |
| Add a remote | `git remote add NAME URL` | `jj git remote add NAME URL` |
| List remotes | `git remote -v` | `jj git remote list` |
| Fetch | `git fetch [REMOTE]` | `jj git fetch [--remote REMOTE]` |
| Push one branch or bookmark | `git push REMOTE BRANCH` | `jj git push -b BOOKMARK [--remote REMOTE]` |
| Push all named work | `git push --all` | `jj git push --all` |
| Push a change without naming it first | no direct form | `jj git push -c REV` |

`jj git push` updates rewritten remote history with force-with-lease style safety. Fetch first if its safety check reports that the remote moved.

## Inspect repository state

| Goal | Git | Jujutsu |
| --- | --- | --- |
| Status | `git status` | `jj st` |
| Current diff | `git diff HEAD` | `jj diff` |
| Diff of one revision | `git diff REV^ REV` | `jj diff -r REV` |
| Diff A to B | `git diff A B` | `jj diff --from A --to B` |
| Show a revision | `git show REV` | `jj show REV` |
| Show a file at a revision | `git show REV:PATH` | `jj file show PATH -r REV` |
| Relevant local history | custom `git log` filters | `jj log` |
| Ancestors of working copy | `git log HEAD` | `jj log -r '::@'` |
| All visible history | `git log --all` | `jj log -r 'all()'` |
| Annotate lines | `git blame PATH` | `jj file annotate PATH` |
| List tracked files | `git ls-files` | `jj file list` |

Agents should add `--no-pager` to commands that may page and `--git` to diffs when unified output is easier to parse.

## Record and refine work

| Goal | Git | Jujutsu |
| --- | --- | --- |
| Add, remove, or modify a file | edit, then `git add` | edit the file; it is snapshotted automatically |
| Finish current work and start a child | `git commit` | `jj commit -m "Message"` |
| Start another empty change | new commit after committing | `jj new -m "Message"` |
| Amend into the parent | `git commit --amend` | `jj squash -u` |
| Move work into an older change | fixup plus autosquash | `jj squash --into REV -u` |
| Split current work interactively | `git add -p` plus commit | `jj split -i` or `jj commit -i` |
| Split by path without an editor | several Git commands | `jj split -m "Message" PATH...` |
| Distribute edits to owning ancestors | fixup commits | `jj absorb` |
| Edit a description | amend or interactive rebase | `jj describe [-r REV] -m "Message"` |

`jj commit` is useful when you want to close one change and receive a new empty working-copy commit. You do not need it merely to save work; the working copy is already a commit.

## Move through and reshape history

| Goal | Git | Jujutsu |
| --- | --- | --- |
| Start work on top of REV | `git switch -c NAME REV` | `jj new REV` |
| Amend an existing revision directly | checkout plus amend | `jj edit REV` |
| Rebase current branch | `git rebase DEST` | `jj rebase -b @ -o DEST` |
| Rebase one revision only | interactive rebase | `jj rebase -r REV -o DEST` |
| Rebase a revision and descendants | `git rebase --onto ...` | `jj rebase -s REV -o DEST` |
| Insert a revision before another | interactive rebase | `jj rebase -r REV -B TARGET` |
| Reorder several revisions | `git rebase -i` | `jj arrange` |
| Cherry-pick | `git cherry-pick REV` | `jj duplicate REV -o DEST` |
| Create a reverting commit | `git revert REV` | `jj revert -r REV -B @` |
| Create a merge | `git merge A` | `jj new @ A` |
| Abandon a change | `git reset --hard` | `jj abandon REV` |
| Empty a change but keep it | reset content | `jj restore --changes-in REV` |
| Restore paths | `git restore PATH...` | `jj restore PATH...` |

Use `jj new REV` to inspect or build on a revision without amending it. Use `jj edit REV` only when you intend subsequent file edits to rewrite that revision.

## Bookmarks and tags

| Goal | Git | Jujutsu |
| --- | --- | --- |
| List branches | `git branch` | `jj bookmark list` |
| Create a branch pointer | `git branch NAME REV` | `jj bookmark create NAME -r REV` |
| Move it forward | `git branch -f NAME REV` | `jj bookmark move NAME --to REV` |
| Move it backward or sideways | `git branch -f NAME REV` | `jj bookmark move NAME --to REV --allow-backwards` |
| Delete it | `git branch -d NAME` | `jj bookmark delete NAME` |
| List tags | `git tag -l` | `jj tag list` |
| Create a tag | `git tag NAME REV` | `jj tag set NAME -r REV` |
| Delete a tag | `git tag -d NAME` | `jj tag delete NAME` |

A bookmark does not move when you create commits on top of it. Move it explicitly before pushing.

## Stash, reflog, and recovery

| Goal | Git | Jujutsu |
| --- | --- | --- |
| Put current work aside | `git stash` | `jj new @-` |
| Return to that work | `git stash pop` | `jj edit SAVED_CHANGE` |
| Inspect repository actions | `git reflog` | `jj op log` |
| Undo the last action | manual reflog recovery | `jj undo` |
| Redo the last undo | manual reflog recovery | `jj redo` |
| Revert an older action | manual reflog recovery | `jj op revert OPERATION` |
| Restore the whole repository view | manual reflog recovery | `jj op restore OPERATION` |
| Inspect one change's old versions | reflog plus commit IDs | `jj evolog -p -r REV` |

## Source

The full upstream mapping is the official [Git command table](https://docs.jj-vcs.dev/latest/git-command-table/).