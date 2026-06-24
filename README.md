# Coding Skills

Personal Codex skills repository for reusable workflows, references, scripts, and assets.

## Layout

```text
skills/
  skill-name/
    SKILL.md
    agents/openai.yaml
    scripts/
    references/
    assets/
external/
  uber-go-guide/
```

Each skill folder should contain a `SKILL.md` with YAML frontmatter:

```yaml
---
name: skill-name
description: What the skill does and when Codex should use it.
---
```

Keep `SKILL.md` focused on the essential workflow. Put detailed reference material in `references/`, reusable commands in `scripts/`, and output templates or media in `assets/`.

## Skills

- `skills/golang-coding/` - Go coding and review guidance based on the Uber Go Style Guide, with split references under `references/src/` so agents load only the relevant docs.

`external/uber-go-guide/` contains a vendored copy of the Uber Go Style Guide from `https://github.com/uber-go/guide`.
