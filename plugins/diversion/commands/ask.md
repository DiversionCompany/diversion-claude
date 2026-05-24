---
description: Investigate AI conversations behind Diversion commits, files, or features. The ask-analyst sub-agent can blame files, walk `dv log`, fetch the linked Claude Code transcripts for any commits it discovers, and synthesize an answer. Fetch and analysis happen inside the sub-agent so raw transcripts stay out of this session's context -- only summarized answers come back.
---

Open or continue an investigation handled by the `ask-analyst` sub-agent.

1. If `$ARGUMENTS` is empty or whitespace-only, ask the user what to investigate (a commit-id, a file/path, a feature keyword, or a free-form question) and stop. Do NOT spawn the analyst with an empty query.

2. Spawn the `ask-analyst` sub-agent via the `Task` tool with prompt:

   > Query: `<$ARGUMENTS verbatim>`. Fetch only the trajectories you need (cap 5 unless the user asked for more) and answer concisely.

   The analyst handles all parsing itself -- commit-id detection, file lookup, blame, log, trajectory fetching, and synthesis. Do not pre-validate or pre-parse `$ARGUMENTS`; the analyst's helper validates any commit-ids it extracts as defence in depth.

3. Show the analyst's response to the user verbatim.

4. **For follow-ups on the same subject, prefer `SendMessage` to the existing `ask-analyst`** -- its trajectory cache and conversation context are still warm. Rules:

   - Address the analyst by **agent id**, not by `name`. Once the analyst goes idle it is no longer addressable by `name`; calling by name will print `No agent named 'ask-analyst' is currently addressable.`
   - When `message` is a plain string, you MUST also pass a one-line `summary` field. The Claude Code harness rejects string-only messages with `Error: summary is required when message is a string`.
   - If the prior analyst is no longer resumable for any reason, fall back to a fresh `Task` call with `subagent_type: ask-analyst`.
   - Never inline raw JSONL into this session's context. The analyst's local cache makes follow-ups cheap (no re-fetch).

The whole point of this command is the isolation: the raw Claude Code transcripts only live inside the sub-agent's context. The main session only ever sees the summarized answers.
