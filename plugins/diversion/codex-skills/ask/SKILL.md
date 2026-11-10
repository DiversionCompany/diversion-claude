---
name: ask
description: Investigate the AI conversations behind commits, files, or features in a Diversion repo or a git repo synced to Diversion. Use when the user runs $diversion:ask, gives a commit id (dv.commit.<digits> or a git SHA) and asks why it changed, or asks why code was written a certain way. A sub-agent reads the linked transcripts, so only a summary reaches this conversation.
---

# Diversion ask

Answer the user's question with an analyst sub-agent. The raw transcripts must stay inside the sub-agent. This conversation sees only the analyst's summary.

`<query>` below is the user's request, exactly as typed, without the `$diversion:ask` mention.

`<dv>` below is the Diversion CLI name. Run `printenv DIVERSION_DV_BIN`: if it prints a name, use it, otherwise use `dv`. Write `<dv>` out at the start of every Diversion command (for example `dvt log -n 5`). The rules that `$diversion:setup-permissions` installs do not match a command that starts with a shell variable or uses a redirect, so Codex would ask for approval.

1. If `<query>` is empty, ask the user what to investigate (a commit id or git SHA, a file or path, a feature, or a free-form question) and stop.

2. **Record the query.** Run:

   ```sh
   <dv> claude-hook track-ask --phase invoked --query '<query>'
   ```

   - Put `<query>` inside single quotes, unchanged. Replace each `'` inside it with `'\''`. Do not trim, rewrite, or move it into a variable.
   - The output has up to two lines: `workspace=<dv|git|none>` and `event_id=<uuid>`. Keep both values. If `event_id` is missing, continue without it.
   - The command needs the Diversion agent and the network. If the sandbox blocks it, ask to run it with more permissions; `$diversion:setup-permissions` stops these prompts.

3. **Pick the analyst instructions** from the `workspace=` value. The files are in this plugin, two directories above this `SKILL.md`:
   - `workspace=dv` -> `agents/ask-analyst.md`
   - `workspace=git` -> `agents/ask-analyst-git.md`
   - `workspace=none` -> this repo is not synced to Diversion, so there are no recorded conversations. Tell the user and stop.
   - No `workspace=` line -> if `.diversion/dv.ws.*` exists at the repo root, use `agents/ask-analyst.md`. Otherwise tell the user you could not confirm that the repo is synced to Diversion, and stop.

4. **Start the analyst** with `spawn_agent`. Set `fork_turns` so that the sub-agent gets none of this conversation. Use this task text, with the absolute path of the chosen file:

   > Read `<absolute path of the analyst file>` and follow it exactly. You run in Codex, so these changes apply:
   > - Read files with `sed -n '<a>,<b>p' <file>` and search with `rg`. These replace the `Read`, `Grep`, and `Glob` tools that the file names, and they are allowed. Read large transcript files in parts.
   > - The Diversion CLI is `<dv>`. Start every Diversion command with that name written out, not with a shell variable.
   > - Add `--coding-agent codex` to the `track-ask --phase completed` command, and run it without the output redirects and `|| true`. Ignore its output.
   > - `dv` commands need the Diversion agent and the network. If the sandbox blocks one, ask to run it with more permissions.
   >
   > Query: `<query>`. EventID: `<event_id, or empty>`. Fetch only the transcripts you need (at most 5 unless the query asks for more) and answer concisely.

   If `spawn_agent` is not available, follow the analyst file yourself with the same changes.

5. Wait for the analyst with `wait_agent`, then show its answer to the user unchanged.

6. For follow-up questions on the same subject, send them to the same analyst with `followup_task`. Its transcript cache is still warm. Start a new analyst only if the old one is gone. Never paste raw transcript lines into this conversation.
