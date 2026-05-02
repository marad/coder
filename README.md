# coder

A tiny wrapper around [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) that runs a self-contained **orchestrator → coder → reviewer** loop on a single coding task, fully non-interactively.

You give it a focused prompt; it implements the change, writes/runs tests, reviews itself, and loops until the reviewer has no blockers.

## How it works

Three subagents are defined in [`agents.json`](./agents.json):

- **orchestrator** — the primary agent. Owns your prompt. Never edits files itself. Plans the work, dispatches the coder, sanity-checks the report, dispatches the reviewer, and loops if there are blockers (capped at 3 cycles).
- **coder** — implements one focused brief at a time. Writes production code, adds/updates tests, runs them, reports back. No scope creep.
- **reviewer** — read-only. Reads the diff and surrounding code, returns findings as `BLOCKER` / `NIT` / `PRAISE` with an `APPROVE | REQUEST_CHANGES` verdict. Calibrated to not escalate style preferences to blockers.

The orchestrator re-runs the coder's tests itself before calling the reviewer — this catches the common failure mode where a subagent claims green tests it didn't actually run.

## Requirements

- [`claude`](https://docs.claude.com/en/docs/claude-code/overview) (the Claude Code CLI) on `PATH`
- `jq` on `PATH`
- `bash`

## Install

Clone the repo somewhere stable, then symlink the script into a directory on your `PATH`:

```bash
git clone https://github.com/marad/coder.git ~/src/coder
ln -s ~/src/coder/coder ~/.local/bin/coder
```

The script resolves its real path via `readlink -f`, so the symlink works from anywhere as long as `agents.json` lives next to the original script.

## Usage

```bash
coder "<prompt>"
echo "<prompt>" | coder
coder -h | --help     # full help, written for both humans and AI callers
```

It operates on the **current working directory** — `cd` into the repo (or subtree) you want it to touch before running it.

### Good prompts

Aim for one focused unit of work with a clear definition of done:

```bash
coder "add a --dry-run flag to scripts/deploy.sh that prints the rsync \
       command instead of running it. Cover with a bats test."

coder "fix the off-by-one in pkg/parse/lexer.go:tokenize when input ends \
       with a backslash. Add a unit test that would fail today."
```

Avoid open-ended prompts (`"clean up the codebase"`) — the orchestrator will pick a narrow interpretation that may not match your intent.

## Output

Progress streams live, prefixed by which agent is talking:

```
[init] session started
ORCHESTRATOR: Reading the task...
[orchestrator → tool: Agent (coder) — implement --dry-run flag]
CODER: Looking at scripts/deploy.sh...
[coder → tool: Edit]
[coder → tool: Bash]
...
[orchestrator → tool: Agent (reviewer) — review the change]
REVIEWER: BLOCKERS: None. NITS: ... VERDICT: APPROVE

=== final ===
<orchestrator's summary>
```

Anything between the last `=== final ===` line and EOF is the canonical result for programmatic consumers.

## Session logs

Every run is logged to `${CODER_LOG_DIR:-~/.local/state/coder}/sessions/<UTC-timestamp>-<pid>/`:

| File             | Contents                                              |
|------------------|-------------------------------------------------------|
| `prompt.txt`     | The prompt as received                                |
| `meta.txt`       | Session id, cwd, start/end time, exit code            |
| `raw.jsonl`      | Full event stream from Claude (one JSON object/line)  |
| `transcript.txt` | The formatted human-readable output                   |

A `sessions/latest` symlink always points to the most recent session, so:

```bash
cat ~/.local/state/coder/sessions/latest/transcript.txt
```

## Environment

| Variable          | Effect                                                                |
|-------------------|-----------------------------------------------------------------------|
| `CODER_LOG_DIR`   | Override the log root (default: `$XDG_STATE_HOME/coder` or `~/.local/state/coder`) |
| `CODER_RAW=1`     | Show raw JSONL events on screen instead of the formatted transcript. Files are written either way. |

## Safety notes

`coder` runs Claude Code with `--permission-mode bypassPermissions`, which means the coder subagent will **edit files and run shell commands without prompting**. This is the price of "no interaction." Implications:

- Run it on repos whose state you're willing to have modified. Uncommitted work may be touched.
- It will not push, open PRs, or commit unless your prompt explicitly tells it to. Even then, prefer reviewing the diff and committing yourself.
- Don't point it at anything you wouldn't trust an unattended script with.

## Exit codes

| Code | Meaning                                                                |
|------|------------------------------------------------------------------------|
| `0`  | Run completed (reviewer approved, or loop cap reached — check transcript) |
| `1`  | Missing `agents.json` next to the script                               |
| `2`  | No prompt supplied (no args and stdin is a tty)                        |
| *    | Any other code is propagated from `claude` itself                      |

## License

[MIT](./LICENSE)
