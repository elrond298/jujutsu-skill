# The Edit Workflow

Use this guide when you prefer to work directly on feature changes instead of the squash workflow, or when a change must land *before* the current one.

The squash workflow (SKILL.md — Essential Workflow) is the recommended default: describe a change, create a scratch `@` on top, work, then `jj squash`. The edit workflow is a leaner alternative when each change is worked on directly.

1. Start a feature change: `jj new -m "only print hello world"`
2. Do the work in it; if that's all you need, you're done
3. To add a change that must come BEFORE the current one:
   `jj new -B @ -m "add more comments"` — inserts before `@` and automatically
   rebases descendant commits (always succeeds; conflicts resolve later)
4. Make that change, then return to the main change with `jj edit <change-id>`
   or `jj next --edit` (edits the child of `@`)

Note: `jj new -B` rebases descendants — change IDs stay stable while commit IDs change.

## Source

[Steve Klabnik's jujutsu tutorial — real-world workflows](https://steveklabnik.github.io/jujutsu-tutorial/real-world-workflows/intro.html)