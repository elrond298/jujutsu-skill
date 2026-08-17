# Running a command across revisions (`jj run`)

Use this guide when you need to apply one command to many commits — formatting, linting, testing, or regenerating files across a stack, or building each commit in isolation.

## What it does

`jj run` checks out each selected revision in an isolated working copy, runs a
command, and amends the revision with the resulting working copy — effectively
"apply this command to every commit", with automatic descendant rebasing.

Typical uses: apply a formatter or pre-commit hook across a stack, regenerate
files, or test each commit in isolation.

Key facts:

- Subprocesses run in temporary working copies (under `.jj/`), so your own
  working copy and files are never touched — you can keep editing while `jj run`
  runs.
- Like all rewrite commands, it refuses immutable/public commits; only
  `--ignore-changes` (no rewrite at all) or `--ignore-immutable` bypass that.
- Except with `--ignore-changes`, `jj run` rewrites the selected commits, so
  commit IDs change. Descendants are rebased to propagate the diff by default;
  `--restore-descendants` keeps descendants' content unchanged instead.

## Options

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

## Examples

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

## Source

Verified against `jj run --help` on jj 0.44.0; background design: [docs.jj-vcs.dev — design/run](https://docs.jj-vcs.dev/v0.44.0/design/run/).