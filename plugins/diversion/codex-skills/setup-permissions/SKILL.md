---
name: setup-permissions
description: Let Codex run the read-only Diversion (dv) commands that $diversion:ask uses without asking each time. Use when the user runs $diversion:setup-permissions or asks to stop the approval prompts for dv commands.
---

# Diversion setup-permissions

Install the plugin's Codex rules file, so Codex runs the read-only `dv` and `dvt` commands of `$diversion:ask` without asking. Commands that change a repo, such as `dv commit`, still ask.

1. The source file is `codex-rules/diversion.rules` in this plugin, two directories above this `SKILL.md`. The target is `rules/diversion.rules` in the Codex home folder: the value of `CODEX_HOME` if it is set, otherwise `~/.codex`. Use absolute paths.

2. If the target exists, compare it with `cmp -s <source> <target>`. If they are the same, tell the user the rules are already installed and stop.

3. Copy the file: `mkdir -p <Codex home>/rules && cp <source> <target>`. The Codex home is outside the workspace, so if the sandbox blocks the command, ask to run it with more permissions. Do not change any other file in `rules/`.

4. Check the file: `codex execpolicy check --rules <target> -- dv log -n 1`. The output must contain `"decision":"allow"`. If it does not, tell the user the check failed and show the output.

5. Tell the user: "Restart Codex for the new rules to take effect."
