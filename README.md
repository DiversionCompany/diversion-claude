# Diversion Plugin for Claude Code and Codex

Diversion plugin for Claude Code and Codex. It captures your Claude Code and Codex conversations and links them to the commits they produced -- so you can later query why any change was made, what was discussed, and how decisions were reached. Works in two flavors of repository:

- **Diversion workspaces** -- the plugin stamps each `dv commit` with a checkpoint id directly.
- **Git repository continuously synced to Diversion** -- the plugin injects a `Diversion-Checkpoint-Id` trailer into each `git commit`, and Diversion's git import wires the trajectory up after the commit is pushed.

## What you get

- **Checkpointed chat history.** Every mutating tool call (Edit, Write, MultiEdit, Bash) records a checkpoint covering the new transcript bytes.
- **Commits stamped with their chat.** In Diversion repos, the plugin attaches the latest checkpoint id to `dv commit`. In git repos continuously synced to Diversion, the plugin appends a `Diversion-Checkpoint-Id` trailer to the commit message; the link materializes once the commit is pushed and Diversion imports it.
- **`dv` as the default VCS in Diversion workspaces.** The plugin tells Claude to use `dv` (not `git`) for commit, branch, merge, etc.
- **Plain `git` in Git repositories continuously synced to Diversion.** No behavior change for you -- just a trailer added under the hood.
- **`/diversion:ask` slash command.** Ask questions about any commit, file, or feature -- the analyst blames the relevant files, picks the commits that matter, and reads each commit's linked chat transcript to answer. Works in both flavors: in Diversion workspaces it blames with `dv` and looks up `dv.commit.*` ids; in git repos synced to Diversion it blames with `git` and looks up commits by SHA.
- **`/diversion:checkpoint-status` slash command.** Print a one-line summary of the current session's checkpoint state.

Failure mode is fail-open: if anything goes wrong (no `dv` on PATH, repo not synced to Diversion, server unreachable) the tool call proceeds unchanged.

## Install

### Prerequisites

- Diversion installed (provides `dv` on your `PATH`). If not installed, follow the install guide at https://docs.diversion.dev/quickstart.
- Launch `claude` from inside a `.diversion/` workspace (or a subdir of one).

### Install via the Claude Code marketplace

Run inside Claude Code:

```
/plugin marketplace add DiversionCompany/diversion-claude
/reload-plugins
/plugin install diversion@diversion
```

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

By default Claude Code prompts before running each `dv` (and, in git workspaces, read-only `git`) subcommand the plugin invokes from `/diversion:ask` and `/diversion:checkpoint-status`. Add the plugin's read-only `dv` and `git` patterns to `~/.claude/settings.json` once:

```
/diversion:setup-permissions
```

The command edits `~/.claude/settings.json` in place, only adds missing entries, and prints which ones were new. Restart Claude Code afterwards.

## Using `/diversion:ask`

`/diversion:ask <query>` answers questions about your repo by combining the code, blame/log metadata, and the Claude Code transcripts that were captured at commit time. The raw transcripts stay inside a sub-agent's context; only the summarized answer comes back to your session. The command auto-detects your workspace: a `.diversion/` workspace routes to the `dv`-based analyst, a `.git/` workspace synced to Diversion routes to the `git`-based analyst.

You can ask about:

- **A specific commit** -- "what was the AI doing in `dv.commit.41508`?" (or a git SHA in a git workspace).
- **A file or path** -- "why was `core/src/api/auth.py` rewritten last month?"
- **A feature or area** -- "how did we end up with the current merge-conflict resolver?"
- **A line range** -- "explain `core/src/sync/engine.py` lines 120-180."
- **A time window** -- "what AI-assisted work landed last week?"

Examples:

```
/diversion:ask dv.commit.41508
/diversion:ask 0123456789abcdef0123456789abcdef01234567
/diversion:ask why did we change the retry logic in core/src/sync/engine.py
/diversion:ask summarize last week's AI work on the auth subsystem
```

In a git workspace, `/diversion:ask` resolves a commit only after it has been pushed to the Diversion-synced remote (the link materializes at import time); commits that are still local-only return "no trajectory".

Follow-up questions on the same subject reuse the analyst's local cache, so they're cheap -- just keep asking.

## Codex

The same plugin works in the Codex CLI. It records your Codex conversations and links them to your commits, like in Claude Code.

### Install

The hooks need a recent `dv`, so update Diversion to the latest version first. Then run:

```
codex plugin marketplace add DiversionCompany/diversion-claude
codex plugin add diversion@diversion
```

### Trust the hooks

Codex does not run plugin hooks until you trust them. Start `codex` in your repo, run `/hooks`, and trust the Diversion hooks. After a plugin update, Codex can ask you to trust them again.

### One-time: stop the `dv` approval prompts

The Codex sandbox blocks the network, so Codex asks before each `dv` command that `$diversion:ask` runs. To allow those read-only commands, run this once and restart Codex:

```
$diversion:setup-permissions
```

It copies the plugin's rules file to `~/.codex/rules/diversion.rules`. Commands that change a repo, such as `dv commit`, still ask.

### Updating

Codex updates the plugin on its own when it starts. To update right away, run `codex plugin marketplace upgrade diversion` and restart Codex.

The update does not change your installed rules file, because it is a copy. After an update, run `$diversion:setup-permissions` again to get the new rules.

### Ask about a commit

Use the `$diversion:ask` skill. It works like `/diversion:ask` above:

```
$diversion:ask dv.commit.41508
$diversion:ask why did we change the retry logic in core/src/sync/engine.py
```

### Notes

- `dv` needs the Diversion agent and the network. When the Codex sandbox blocks a `dv` command, approve it.
- In a Diversion workspace that has no `.git` directory, trust the folder when Codex asks.

## Your recorded sessions

While the plugin is on, it uploads the transcript of each Claude Code or Codex session to Diversion and links it to your commits. The transcripts belong to your Diversion account, no matter which computer, Claude account, or ChatGPT account recorded them.

### Stop recording

The plugin is installed per computer. On each computer where you use it, run inside Claude Code:

```
/plugin disable diversion@diversion
```

To remove it completely, use `/plugin uninstall diversion@diversion`. Then restart Claude Code, so that sessions that are already open stop recording too.

In Codex, run `codex plugin remove diversion@diversion`, then restart Codex.

### Delete recorded sessions

Open **Settings > AI** in the [Diversion web app](https://app.diversion.dev/settings/ai) and use **Delete Claude Code session transcripts**:

- Choose **All repositories**, or pick up to 10 specific repositories.
- The deletion removes the transcripts. Your commits stay, but no longer show a transcript. Your files do not change.
- It deletes the sessions of your Diversion account only. Other users' sessions do not change.
- Stop recording first. Otherwise the plugin records your next sessions again.

This page is available on Diversion cloud. On a self-hosted install, contact support.

## Support

For questions or issues with the plugin:

- **Chat with us** in the [Diversion app](https://app.diversion.dev) (desktop or web) -- look for the chat widget
- **Email** [support@diversion.dev](mailto:support@diversion.dev?subject=Claude%20Code%20Plugin%20Support)
- **Bug reports** -- open an issue on [GitHub](https://github.com/DiversionCompany/diversion-claude/issues)
