---
description: Show the Diversion plugin checkpoint state for the current session.
---

Print a summary of the Diversion plugin state.

1. Read `${CLAUDE_PROJECT_DIR}/.diversion/diversion-claude/state-<session_id>.json` for the current session. If missing, report that no checkpoints have been recorded yet for this session.
2. Read and show these env vars:
   - `DIVERSION_DV_BIN` (default `dv`)
   - `DIVERSION_AGENT_TYPE` (default `claude-code`)
   - `DIVERSION_CHECKPOINT_TIMEOUT_MS` (default `5000`)
   - `DIVERSION_CHECKPOINT_DEBUG`
3. Resolve the `dv` binary path (`which $DIVERSION_DV_BIN` / `which dv`) and run `dv status -nowait` once; report whether it exited 0 so the user can tell if the CLI prerequisite is met in this cwd.
4. Print a one-line summary: `agent_session_id`, `transcript_lines`, `last_checkpoint_id`, `last_transcript_bytes`, `updated_at`.

Do not mutate any files. If the state file exists but is unreadable, report the path and the parse error.
