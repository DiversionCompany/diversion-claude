# Diversion Claude Code Plugin

Diversion plugin for Claude Code. It records the **trajectory** of a coding session -- the ordered chain of chat turns, tool inputs, and commits -- and ties it back to the Diversion commits the session produced.

## What you get

- **Checkpointed chat history.** Every mutating tool call (Edit, Write, MultiEdit, Bash) records a checkpoint covering the new transcript bytes.
- **Commits stamped with their chat.** When you run `dv commit`, the plugin attaches the latest checkpoint id, so the commit is permanently linked to the conversation that produced it.
- **`dv` as the default VCS.** In a Diversion workspace, the plugin tells Claude to use `dv` (not `git`) for commit, branch, merge, etc.
- **`/diversion:ask` slash command.** Ask questions about any commit, file, or feature -- the analyst blames the relevant files, picks the commits that matter, and reads each commit's linked chat transcript to answer.
- **`/diversion:checkpoint-status` slash command.** Print a one-line summary of the current session's checkpoint state.

Failure mode is fail-open: if anything goes wrong (no `dv` on PATH, not in a Diversion repo, server unreachable) the tool call proceeds unchanged.

## Install

### Prerequisites

- `dv` on your `PATH`, version `0.9.910` or later. Confirm with `dv --version`.
- Launch `claude` from inside a `.diversion/` workspace (or a subdir of one).

### Install via the Claude Code marketplace

Run inside Claude Code:

```
/plugin marketplace add DiversionCompany/diversion-claude
/plugin install diversion@diversion
```

The marketplace clones the repo with your existing GitHub credentials (SSH key, `gh auth`, or `git credential` helper), so this works for the private `DiversionCompany/diversion-claude` repo as long as you have clone access.

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

### One-time: silence the `dv` Bash permission prompts

By default Claude Code prompts before running each `dv` subcommand the plugin invokes from `/diversion:ask` and `/diversion:checkpoint-status`. Add the plugin's read-only `dv` patterns to `~/.claude/settings.json` once:

```
/diversion:setup-permissions
```

The command edits `~/.claude/settings.json` in place, only adds missing entries, and prints which ones were new. Restart Claude Code afterwards.

## Using `/diversion:ask`

`/diversion:ask <query>` answers questions about your repo by combining the code, `dv` blame/log metadata, and the Claude Code transcripts that were captured at commit time. The raw transcripts stay inside a sub-agent's context; only the summarized answer comes back to your session.

You can ask about:

- **A specific commit** -- "what was the AI doing in `dv.commit.41508`?"
- **A file or path** -- "why was `core/src/api/auth.py` rewritten last month?"
- **A feature or area** -- "how did we end up with the current merge-conflict resolver?"
- **A line range** -- "explain `core/src/sync/engine.py` lines 120-180."
- **A time window** -- "what AI-assisted work landed last week?"

Examples:

```
/diversion:ask dv.commit.41508
/diversion:ask why did we change the retry logic in core/src/sync/engine.py
/diversion:ask summarize last week's AI work on the auth subsystem
```

Follow-up questions on the same subject reuse the analyst's local cache, so they're cheap -- just keep asking.
