---
description: Investigate AI conversations behind commits, files, or features. A sub-agent blames files, walks the commit log, fetches the linked Claude Code transcripts for any commits it discovers, and synthesizes an answer. Works in both native Diversion (`.diversion/`) workspaces and `.git/` repos continuously synced to Diversion. Fetch and analysis happen inside the sub-agent so raw transcripts stay out of this session's context -- only summarized answers come back.
---

Open or continue an investigation handled by the appropriate ask-analyst sub-agent (`ask-analyst` for native Diversion workspaces, `ask-analyst-git` for `.git/` workspaces synced to Diversion).

1. If `$ARGUMENTS` is empty or whitespace-only, ask the user what to investigate (a commit-id or git SHA, a file/path, a feature keyword, or a free-form question) and stop. Do NOT spawn the analyst with an empty query.

2. **Record the verbatim query.** Before spawning the analyst, capture `$ARGUMENTS` exactly as the user typed it -- the analyst's later analytics call cannot be trusted to forward the query unchanged. Run:

   ```sh
   dv claude-hook track-ask --phase invoked --query '<$ARGUMENTS verbatim>' 2>/dev/null
   ```

   Rules:
   - Wrap `$ARGUMENTS` in single quotes so `$`, `` ` ``, `\`, and `"` pass through unchanged to argv. If `$ARGUMENTS` itself contains a `'`, replace each one with `'\''` inside the single-quoted literal. Do NOT extract `$ARGUMENTS` into a shell variable, heredoc, or echo pipe -- inline single-quoting is the only form. Do NOT paraphrase, trim, or normalise.
   - Stdout is up to two lines: `workspace=<dv|git|none>` (always emitted) and `event_id=<uuid>` (only when the audit log succeeded). Capture **both**: use `workspace=` for routing in step 3, and reuse `<uuid>` in step 4. If `event_id` is missing (network failure, unauthenticated session), continue without an id.
   - Failure is non-fatal: even on a blank stdout, continue to step 3 (treat it as a missing `workspace=` line there).

3. **Pick the analyst by the `workspace=` value from step 2.** `.git/` presence alone does NOT mean the repo is synced to Diversion -- only the plugin's resolved verdict does. Route strictly:
   - `workspace=dv` -> native Diversion workspace -> use `ask-analyst`.
   - `workspace=git` -> a `.git/` repo the plugin **confirmed** is synced to a Diversion repo -> use `ask-analyst-git`.
   - `workspace=none` -> the repo is NOT linked to Diversion (a plain git repo without Diversion sync, or not a repo at all). There are no captured AI conversations to query: tell the user this repo is not synced to Diversion, so `/diversion:ask` has nothing to look up, and STOP. Do NOT spawn an analyst, and do NOT fall back to `ask-analyst-git` just because a `.git/` directory exists.
   - **`workspace=` line missing** (track-ask produced no stdout -- e.g. the Diversion CLI not on PATH): fall back to a filesystem check with the `Glob` tool -- if `.diversion/dv.ws.*` exists at the repo root use `ask-analyst`; otherwise tell the user you could not confirm this repo is synced to Diversion and STOP.

   Call the chosen agent `<analyst>` below.

4. Spawn the chosen `<analyst>` sub-agent via the `Task` tool (`subagent_type: <analyst>`) with prompt:

   > Query: `<$ARGUMENTS verbatim>`. EventID: `<uuid captured in step 2, or empty if none>`. Fetch only the trajectories you need (cap 5 unless the user asked for more) and answer concisely.

   The analyst handles all parsing itself -- commit-id / git-SHA detection, file lookup, blame, log, trajectory fetching, and synthesis. Do not pre-validate or pre-parse `$ARGUMENTS`; the analyst's helper validates any ids it extracts as defence in depth.

5. Show the analyst's response to the user verbatim.

6. **For follow-ups on the same subject, prefer `SendMessage` to the existing analyst** -- its trajectory cache and conversation context are still warm. Rules:

   - Address the analyst by **agent id**, not by `name`. Once the analyst goes idle it is no longer addressable by `name`; calling by name will print `No agent named '<analyst>' is currently addressable.`
   - When `message` is a plain string, you MUST also pass a one-line `summary` field. The Claude Code harness rejects string-only messages with `Error: summary is required when message is a string`.
   - If the prior analyst is no longer resumable for any reason, fall back to a fresh `Task` call with `subagent_type: <analyst>` (the same kind chosen in step 3).
   - Never inline raw JSONL into this session's context. The analyst's local cache makes follow-ups cheap (no re-fetch).

The whole point of this command is the isolation: the raw Claude Code transcripts only live inside the sub-agent's context. The main session only ever sees the summarized answers.
