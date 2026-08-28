# Embedded agent skills and rules

Agent workflows and optional Cursor rules for MCU firmware in C/C++. Aimed at bare-metal and RTOS work. Most focus was given to C, C++ coverage is currently
lacking and will be expanded in the future.

Install them into a firmware repo so the agent there follows the same review, grilling, simplification, and memory procedures.

## Install

Any agent that the [skills CLI](https://github.com/vercel-labs/skills) supports:

```bash
npx skills add iharhl/embedded-agent-skills
```

Pick which skills to take, and which agents to install them on. That copies each skill folder into the path that agent reads (`.claude/skills/`, `.agents/skills/`, `.opencode/skills/`, and so on).

**Claude Code** (skills only)

```text
/plugin marketplace add iharhl/embedded-agent-skills
/plugin install embedded-agent-skills@embedded-agent-skills
```

**Codex** (skills only)

```bash
codex plugin marketplace add iharhl/embedded-agent-skills
codex plugin add embedded-agent-skills@embedded-agent-skills
```

**Cursor**

Clone or fetch this repo, then copy. Do not paste skill bodies into rules.

```bash
git clone https://github.com/iharhl/embedded-agent-skills.git
mkdir -p your-firmware/.cursor/skills <your-firmware>/.cursor/rules
cp -R embedded-agent-skills/skills/. <your-firmware>/.cursor/skills/
cp embedded-agent-skills/.cursor/rules/commenting.mdc your-firmware/.cursor/rules/
```

Take **commenting**. The **c-conventions** should be tailored per-project.

```bash
cp embedded-agent-skills/.cursor/rules/c-conventions.mdc <your-firmware>/.cursor/rules/
```

**OpenCode** (skills only)

Copy skills into `.opencode/skills/` (project) or `~/.config/opencode/skills/` (global).

```bash
git clone https://github.com/iharhl/embedded-agent-skills.git
mkdir -p your-firmware/.opencode/skills
cp -R embedded-agent-skills/skills/. <your-firmware>/.opencode/skills/
```

Claude, Codex, and OpenCode ignore `.cursor/rules/`. Paste rule bodies into that project's `AGENTS.md` if you want the same constraints there.

## Invocation

Skills are slash-only. Each skill sets `disable-model-invocation: true`. Codex also has `agents/openai.yaml` with `allow_implicit_invocation: false`.

| Host | Example |
| --- | --- |
| Cursor | `/review-embedded` |
| Codex | `$review-embedded` |
| Claude Code | `/embedded-agent-skills:review-embedded` |

Before judging firmware, the skills read `CLAUDE.md`, `AGENTS.md`, and `.cursor/rules/` if those exist.

In Cursor, the `.mdc` rules apply when matching `*.c` / `*.h` files are in context. This library's own `AGENTS.md` is for maintaining the library, not firmware style.

## Layout

```text
skills/                  # source of truth (all agents)
  review-embedded/
  grilling-embedded/
  code-simplification-embedded/
  memory-optimization-embedded/
.cursor/rules/           # Cursor rules (.mdc); optional
.claude-plugin/          # Claude Code plugin + marketplace
.codex-plugin/           # Codex plugin
.agents/plugins/         # Codex marketplace
```

## What is here

**Rules**

- **commenting.** Portable default. `/* */` on non-trivial logic. Why, not restating the next line, not leftover fix narrative. `*.c` / `*.h`.
- **c-conventions.** Opinionated house style. Quote includes vs compiler headers, host-test stub include order, `FILENAME_H` guards, `ptr == 0`, unsigned `U`.

**Skills**

- **review-embedded.** Read-only review of a firmware diff. Sanity, defects, ISR/main races, qualitative resource signals, duplication, peripheral edge cases, docs. Silicon-facing changes need the datasheet or reference manual. `/review-embedded` or `$review-embedded`.
- **grilling-embedded.** Architecture interview before code, scoped to a named feature unless you ask for the whole product. Rounds of frontier questions with a recommended answer on each, then a decision log. `/grilling-embedded` or `$grilling-embedded`.
- **code-simplification-embedded.** Clarify working C firmware without changing behavior, including timing, footprint, register order, and errata workarounds. `/code-simplification-embedded` or `$code-simplification-embedded`.
- **memory-optimization-embedded.** Profile the image with size/map/disassembly, apply safe RAM and Flash cuts on the requested feature, and propose neighboring or whole-image wins. `/memory-optimization-embedded` or `$memory-optimization-embedded`.

These are firmware-specific. For general agent workflow (how/why a subsystem works, review, verification), [pstack](https://github.com/cursor/plugins/tree/main/pstack) is a good companion in Cursor (`/add-plugin pstack`).

## Add a skill

1. Create `skills/my-skill/SKILL.md` with `name` and `description` frontmatter. `name` must match the folder.
2. Keep the body short. Put long tables in `references/`.
3. For slash-only skills, set `disable-model-invocation: true` in the frontmatter and add `agents/openai.yaml` with `allow_implicit_invocation: false`.
4. Link it from this README.

Add a rule as `.cursor/rules/my-rule.mdc` with `description` and either `alwaysApply: true` or `globs`. See `AGENTS.md` for the split between rules and skills.

## Acknowledgements

`grilling-embedded` is adapted from Matt Pocock's [grill-me / grilling](https://github.com/mattpocock/skills/tree/main) skills.

`code-simplification-embedded` is adapted from Addy Osmani's [code-simplification](https://github.com/addyosmani/agent-skills/tree/main) skill.

## License

MIT, see `LICENSE`.
