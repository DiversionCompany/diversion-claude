---
name: setup-permissions
description: Let Codex run Diversion (dv) commands outside the sandbox without asking. Use when the user runs $diversion:setup-permissions, asks to stop the approval prompts for dv commands, or a dv command fails because the sandbox blocked the local Diversion agent.
---

# Diversion setup-permissions

Install the plugin's Codex rules file, so Codex runs every `dv` command outside the sandbox without asking. `dv` talks to the local Diversion agent over the network, and the sandbox blocks the network. The rules also cover commands that change a repo, such as `dv commit` and `dv merge`. Codex still asks before commands that delete data or call the raw API.

1. The source file is `codex-rules/diversion.rules` in this plugin, two directories above this `SKILL.md`. The target is `rules/diversion.rules` in the Codex home folder: the value of `CODEX_HOME` if it is set, otherwise `~/.codex`. Use absolute paths.

2. **Pick the CLI name.** Run `printenv DIVERSION_DV_BIN`. The plugin runs that name instead of `dv` when it is set, but the rules allow only `dv` and `dvt`. `<name>` below and `<print rules>`, the command that prints the rules to install, depend on the output:
   - `dvt`: `<name>` is `dvt`, and `<print rules>` is `sed 's|pattern = \["dv"|pattern = [["dv", "dvt"]|' <source>`. Keep `dv` in each rule, because Codex rejects a rules file whose examples, which use `dv`, match no rule.
   - Nothing, or `dv`: `<name>` is `dv`, and `<print rules>` is `cat <source>`.
   - Any other value: same as nothing. Also tell the user that the rules do not cover `DIVERSION_DV_BIN`, so Codex still asks before each command that starts with that value.

3. If the target exists, compare it with `<print rules> | cmp -s - <target>`. If they are the same, tell the user the rules are already installed and stop.

4. Install the rules: `mkdir -p <Codex home>/rules && <print rules> > <target>`. The Codex home is outside the workspace, so if the sandbox blocks the command, ask to run it with more permissions. Do not change any other file in `rules/`.

5. Check the file: `codex execpolicy check --rules <target> -- <name> log -n 1`. The output must contain `"decision":"allow"`. If it does not, tell the user the check failed and show the output.

6. Tell the user: "Codex now runs `<name>` commands without asking, including `<name> commit`. It still asks before `<name> repo -d`, `<name> workspace delete`, `<name> obliterate`, `<name> prune`, and `<name> api`. Restart Codex for the new rules to take effect."
