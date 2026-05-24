# diversion

Diversion plugin for Claude Code. It captures your Claude Code conversations and links them to the Diversion commits they produced -- so you can later query why any change was made, what was discussed, and how decisions were reached.

How it does it today:

1. On `SessionStart`, if the cwd contains a `.diversion/` directory, the plugin injects an `additionalContext` block telling Claude to use `dv` for all VCS operations (commit, branch, merge, etc.) instead of `git`. This makes the next two steps actually fire.
2. Before every mutating tool call (Edit, Write, MultiEdit, Bash) the plugin invokes `dv claude-hook pre-tool-use`. On the first turn that registers an agent session via the in-process Diversion API; on every subsequent turn it appends the new transcript bytes and creates a checkpoint covering the delta.
3. When the upcoming tool call is `dv commit` (or `dvt commit` / `python3 scripts/dvx.py commit`), the plugin rewrites the command to include `--checkpoint-id <latest_id>` via Claude Code's `hookSpecificOutput.updatedInput`, stamping the commit with the chat state that produced it. The server links the commit back to the checkpoint atomically (see `dv commit-transcript <commit_id>` for the readback).
4. Provides query capabilities over the recorded trajectory (`/diversion:checkpoint-status`, `/diversion:ask <query>`). The `/diversion:ask` analyst can answer questions about specific commits, but also about files or features -- it blames the relevant files, picks the commits that matter, and pulls each commit's linked transcript on demand.

Failure mode is intentionally **fail-open**: any error (CLI missing, not in a Diversion repo, server down, parse failure) lets the tool call proceed unchanged. The `dv commit` rewrite only fires when a checkpoint id was successfully obtained.

The hooks are implemented inside the `dv` binary as hidden subcommands (`dv claude-hook session-start`, `pre-tool-use`, `post-tool-use`, `session-end`, `fetch-trajectory`). The plugin itself is now a thin manifest plus markdown - no Python required on the host.

## Install

### Prerequisites

- A `dv` (or `dvt`) binary on your `PATH`, version `0.9.910` or later. Confirm with `dv --version`. Earlier `dv` releases will exit with `unknown subcommand: claude-hook` and the hook will fail-open (every tool call still works, but no checkpoints are recorded).
- Launch `claude` from inside a `.diversion/` workspace (or a subdir of one).

### Install via the Claude Code marketplace

Run inside Claude Code:

```
/plugin marketplace add DiversionCompany/diversion
/plugin install diversion@diversion
```

The marketplace clones the repo with your existing GitHub credentials (SSH key, `gh auth`, or `git credential` helper), so this works for the private `DiversionCompany/diversion` repo as long as you have clone access.

After install, quit Claude Code (`/quit` or Ctrl-D) and relaunch from the repo you want to use it in:

```bash
cd path/to/your/diversion/repo
claude
```

### Updating

```
/plugin marketplace update diversion
/plugin install diversion@diversion
```

Then relaunch `claude`.

## Verify it's running

After your first tool call in a fresh session:

```bash
ls -la .diversion/diversion-claude/
cat .diversion/diversion-claude/state-*.json
```

The `state-<session_id>.json` file should contain `agent_session_id` and `last_checkpoint_id`. Inside Claude Code, `/diversion:checkpoint-status` prints the same summary.

To confirm the commit linkage, run any `dv commit` from inside Claude Code, then:

```bash
dv commit-transcript <commit_id>
```

The response's `checkpoint.id` matches `last_checkpoint_id` from the state file.

## Configure

| Env var | Required | Default | Purpose |
|---|---|---|---|
| `DIVERSION_DV_BIN` | No | `dv` | Override the `dv` binary path (e.g. `dvt` for the staging CLI). Honored by every shell-out in this plugin: `hooks/hooks.json` invokes `dv claude-hook ...` for each event, and the `/diversion:ask` slash command + analyst use the same form. Supported shapes: a bare command on `PATH` (`dv`, `dvt`), an absolute path (`/opt/dv`), or a multi-token wrapper (`python3 scripts/dvx.py`). The form must be left **unquoted** in scripts so the shell word-splits multi-token wrappers correctly. |
| `DIVERSION_AGENT_TYPE` | No | `claude-code` | Value used as the agent-type on session create. |
| `DIVERSION_CHECKPOINT_TIMEOUT_MS` | No | `15000` | Per-coreapi-call timeout in milliseconds for PreToolUse. |
| `DIVERSION_TRAJECTORY_FETCH_TIMEOUT_MS` | No | `30000` | Per-call timeout for `dv claude-hook fetch-trajectory`. |
| `DIVERSION_CHECKPOINT_DEBUG` | No | `0` | `1` enables debug logging from the hooks (written to the dv log facility `claudehook`). |
| `DIVERSION_TRAJECTORY_DEBUG` | No | `0` | `1` adds extra debug logging from the trajectory fetcher. |

## Files

- `.claude-plugin/plugin.json` - manifest.
- `hooks/hooks.json` - registers SessionStart, PreToolUse (matcher `Edit|Write|MultiEdit|Bash`), PostToolUse (matcher `Bash`), and SessionEnd. Each entry shells out to a hidden `dv claude-hook ...` subcommand so staging users (`DIVERSION_DV_BIN=dvt`) hit `dvt`.
- `agents/ask-analyst.md` - sub-agent that investigates AI conversations linked to commits, files, or features. It can blame files with `dv annotate`, widen with `dv log` / `dv show`, fetch up to 5 trajectories (cached locally) per query, and holds the raw JSONL inside its own context while returning only summarized answers.
- `commands/checkpoint-status.md` - `/diversion:checkpoint-status` slash command.
- `commands/ask.md` - `/diversion:ask <query>` slash command -- investigates AI conversations behind commits, files, or features.

## Verify locally

The hook implementation lives in `client/pkg/claudehook/`. Unit tests:

```
cd client && go test ./pkg/claudehook/...
```

Smoke-test the SessionStart hook against a real Diversion workspace (cwd must be inside one):

```bash
echo '{"cwd":"'"$PWD"'"}' | dv claude-hook session-start
```

Smoke-test the PreToolUse hook (creates a session + checkpoint, returns a rewritten command on stdout):

```bash
echo '{"session_id":"demo","transcript_path":"/tmp/t.jsonl","tool_name":"Bash","tool_input":{"command":"dv commit -m hi"},"cwd":"'"$PWD"'"}' \
  | DIVERSION_CHECKPOINT_DEBUG=1 CLAUDE_PROJECT_DIR="$PWD" \
    dv claude-hook pre-tool-use
```

Inspect `.diversion/diversion-claude/state-demo.json` for the persisted `agent_session_id` / `last_checkpoint_id`, then read it back from the server:

```
dv commit-transcript <commit_id_from_a_real_dv_commit>
```
