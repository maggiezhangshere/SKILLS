# Skills

Personal Codex skills repository.

This repository stores reusable skills under the `skills/` directory. Each skill
is self-contained and follows this layout:

```text
skills/
  <skill-name>/
    SKILL.md
    references/
```

`SKILL.md` is the entry point. Supporting docs, checklists, templates, or other
reference material should live beside it, usually under `references/`.

## Available Skills

| Skill | Purpose |
| --- | --- |
| `goal-driven-review` | Reviews code by first recovering the intended design goal, then checking whether the implementation actually achieves it before judging correctness, reliability, and maintainability. |

## Adding A Skill

Add each new skill in its own directory:

```text
skills/<skill-name>/SKILL.md
```

Keep shared context inside that skill directory instead of relying on external
files. If the skill needs examples, checklists, templates, or deeper guidance,
place them under `skills/<skill-name>/references/`.

After adding a skill, update the table above so the repository stays usable as
an index.
