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

- `skills/golang-coding/` - Full Go coding and review skill based on the Uber Go Style Guide, with concrete bad/good examples in `SKILL.md` and source split references under `references/src/`.

`external/uber-go-guide/` contains a vendored copy of the Uber Go Style Guide from `https://github.com/uber-go/guide`.
