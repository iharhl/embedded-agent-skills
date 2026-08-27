# This repo

This is a library of Cursor rules and agent skills for MCU firmware.

## Where things go

| Kind | Path | Use when |
| --- | --- | --- |
| Skill | `.agents/skills/<skill-name>/SKILL.md` | A workflow the agent should follow. |
| Codex policy | `.agents/skills/<skill-name>/agents/openai.yaml` | Slash-only equivalent for Codex. |
| Rule | `.cursor/rules/<name>.mdc` | Standing constraints in Cursor while editing matching files. Codex does not read this path. |
| Extra docs | `references/` next to that skill's `SKILL.md` | Details the agent should open only when needed. |

Skill folder name, `name:` in frontmatter, and `/skill-name` in chat must match. Lowercase, digits, hyphens. No consecutive hyphens.

Cursor also loads `.cursor/skills/`. Do not put a second copy there. `.agents/skills/` is the only skills tree so Codex and Claude Code can pick the same files up from a clone.

Codex ignores `.cursor/rules/`. Firmware C style for Codex belongs in that project's `AGENTS.md`, copied from the `.mdc` bodies.

## Rules vs skills

Put it in a **rule** if it should constrain everyday edits (C style, comments). Keep each rule to one concern and under 50 lines.

Put it in a **skill** if it is a procedure, such as reviewing a diff, grilling a design, or simplifying working firmware.

- `alwaysApply: true` only for constraints that belong in every firmware session. `commenting` and `c-conventions` use `globs` on `*.c` / `*.h`; they are not always-on. Link flags, libc policy, and allocator/`NULL` exceptions stay in the firmware tree.
- Omit `disable-model-invocation` so the agent can attach a skill from context. Set it to `true` only for slash-only playbooks. `review-agent-embedded`, `grilling-embedded`, `code-simplification-embedded`, and `memory-optimization-embedded` are slash-only. Each has `agents/openai.yaml` with `allow_implicit_invocation: false`.

## Writing a skill

1. Frontmatter `description` is third person, says what it does and when to use it, and includes the words people actually type.
2. `SKILL.md` body stays under 500 lines. Link one level down to `references/`.
3. Infer a firmware tree from `CLAUDE.md`, `AGENTS.md`, and `.cursor/rules/` if they exist. Follow those over the skill when they conflict.
4. After adding a skill or rule, list it in `README.md`. Slash-only skills also need `agents/openai.yaml`.
