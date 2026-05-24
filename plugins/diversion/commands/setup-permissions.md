---
description: Add the Diversion plugin's required Bash permission patterns to ~/.claude/settings.json so dv calls stop re-prompting.
---

Add the Diversion plugin's required `Bash(dv ...)` permission patterns to the user's Claude Code settings.

1. Use the `Read` tool on the file at the absolute path `~/.claude/settings.json` (expand `~` to `$HOME` first by reading the env var from Bash with `echo "$HOME/.claude/settings.json"` -- do not use `~` in the Read path). If the file does not exist, treat the starting content as `{"permissions": {"allow": []}}`.

2. Parse the file as JSON. If parsing fails, stop and report the parse error along with the file path -- do not overwrite a broken file.

3. Ensure `permissions.allow` (an array) contains every one of these entries. Add only the missing ones; never duplicate, never reorder, never remove existing entries:
   - `Bash(dv claude-hook:*)`
   - `Bash(dv annotate:*)`
   - `Bash(dv log:*)`
   - `Bash(dv show:*)`
   - `Bash(dv diff:*)`
   - `Bash(dv status:*)`
   - `Bash(dv branch:*)`

   If `permissions` or `permissions.allow` is missing, create the minimum nested structure to hold the array. Preserve every other key in the file untouched.

4. If no patterns needed to be added (every required entry was already present), skip the write entirely and proceed to step 5. Otherwise use the `Write` tool to save the modified JSON back to the same path with 2-space indentation and a trailing newline. Preserve every key and value -- whitespace and indentation may change as a side effect of re-serializing, but no unrelated keys or values are dropped or modified.

5. Print a short summary in this exact shape:

   ```text
   Added: <comma-separated newly-added patterns, or "none">
   Already present: <comma-separated patterns that were already in the file, or "none">
   ```

6. Tell the user: "Restart Claude Code for the new permission patterns to take effect."

Notes:

- Do not touch project-local `.claude/settings.json` -- the user-level file is the right place because the plugin is installed globally.
- Do not mutate any other file. Read-only outside of `~/.claude/settings.json`.
