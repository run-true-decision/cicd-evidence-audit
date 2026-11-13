# cicd-evidence-audit

Audit CI/CD merge evidence and retrofit five controls: source comparison, local preflight, test collection, required CI results, and workflow effect. The skill maps the evidence behind a merge decision, reports gaps, and guides small, reviewable changes.

## Install on claude.ai or the Claude Desktop app

1. Turn on code execution in Settings → Capabilities.
2. Download [cicd-evidence-audit.zip](https://github.com/run-true-decision/cicd-evidence-audit/releases/latest/download/cicd-evidence-audit.zip) from the latest release.
3. Go to Customize → Skills → "+" → "Create skill" → "Upload a skill" and upload the zip.

## Install in Claude Code

Download the zip linked above. Unzip it so the folder is `~/.claude/skills/cicd-evidence-audit/`. Then type `/cicd-evidence-audit`.

## Limits

The skill is guidance for an audit. It does not change any repository's CI by itself, and you must review its suggestions.

## License

MIT.
