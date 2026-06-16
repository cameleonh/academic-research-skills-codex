# Cameleonh Codex Notes

This fork tracks the Codex-native Academic Research Skills package from
`Imbad0202/academic-research-skills-codex`.

## Install Or Update

Use this fork as the Codex skill source:

```bash
python3 "$HOME/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo cameleonh/academic-research-skills-codex \
  --ref main \
  --path skills/academic-research-suite \
  --method git
```

On Windows, run the same installer with the local Python executable:

```powershell
python "C:\Users\honey\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py" `
  --repo cameleonh/academic-research-skills-codex `
  --ref main `
  --path skills/academic-research-suite `
  --method git
```

After installation, open a new Codex conversation and invoke:

```text
Use $academic-research-suite ...
```

## Use With Paper Writing Mode

`paper-writing-mode` should keep project-specific Korean manuscript work,
journal-target constraints, advisor notes, local result checks, and Obsidian
project routing.

Use `$academic-research-suite` for larger ARS workflows:

- research question refinement
- systematic literature review planning
- end-to-end research-to-paper pipeline staging
- simulated peer review
- citation or integrity gate checks
- experiment planning or reproducibility review

## Attribution And License

Original upstream: `Imbad0202/academic-research-skills-codex`

Original Claude sibling: `Imbad0202/academic-research-skills`

Copyright belongs to Cheng-I Wu. The repository license is CC BY-NC 4.0.
Preserve attribution and keep use/adaptation within the noncommercial license
terms unless a separate license is obtained.
