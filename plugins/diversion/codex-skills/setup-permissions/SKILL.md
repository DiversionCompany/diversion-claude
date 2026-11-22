---
name: setup-permissions
description: Let Codex run the read-only Diversion (dv) commands that $diversion:ask uses without asking each time. Use when the user runs $diversion:setup-permissions or asks to stop the approval prompts for dv commands.
---

# Diversion setup-permissions

Install the plugin's Codex rules file, so Codex runs the read-only `dv` commands of `$diversion:ask` without asking. Commands that change a repo, such as `dv commit`, still ask.

1. The source file is `codex-rules/diversion.rules` in this plugin, two directories above this `SKILL.md`. The target is `rules/diversion.rules` in the Codex home folder: the value of `CODEX_HOME` if it is set, otherwise `~/.codex`. Use absolute paths.

2. **Pick the CLI name.** Run `printenv DIVERSION_DV_BIN`. If it prints a name, `$diversion:ask` runs that name instead of `dv`, so the rules must allow it too. `<name>` below is that name, or `dv` if nothing is printed. If `<name>` has any character other than letters, digits, `.`, `_`, `-`, and `/`, tell the user that `DIVERSION_DV_BIN` is not a valid command name and stop.

   `<print rules>` below is the command that prints the rules to install:
   - `<name>` is `dv`: `cat <source>`
   - Any other `<name>`: `sed 's|pattern = \["dv",|pattern = [["dv", "<name>"],|' <source>`. Keep `dv` in each rule, because Codex rejects a rules file whose examples, which use `dv`, match no rule.

3. If the target exists, compare it with `<print rules> | cmp -s - <target>`. If they are the same, tell the user the rules are already installed and stop.

4. Install the rules: `mkdir -p <Codex home>/rules && <print rules> > <target>`. The Codex home is outside the workspace, so if the sandbox blocks the command, ask to run it with more permissions. Do not change any other file in `rules/`.

5. Check the file: `codex execpolicy check --rules <target> -- <name> log -n 1`. The output must contain `"decision":"allow"`. If it does not, tell the user the check failed and show the output.

6. Tell the user: "Restart Codex for the new rules to take effect."
