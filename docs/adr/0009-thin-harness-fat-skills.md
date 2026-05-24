# ADR-0009: Thin harness, fat skills

**Status:** Accepted
**Landed:** v0.1 (philosophy); formalized in `docs/ethos/THIN_HARNESS_FAT_SKILLS.md`

## Context

GBrain ships agent-facing behavior. The choice is where the "how to do
brain ops" knowledge lives:

- **In compiled code** — typed, testable, but every behavior change is
  a release and every fork has to fork the binary.
- **In markdown files the agent reads** — slower per-call but infinitely
  forkable, user-editable, no release cycle for skill tweaks, and the
  same files work across CLI, MCP, and any future transport.

The first option treats the agent as a dumb runtime. The second treats
the agent as a reader and the skill markdown as the program.

## Decision

The CLI binary (`gbrain`) is intentionally small: it implements ~47
operations and the storage layer, and that's it. The reasoning about
*how to combine those operations into useful work* lives in markdown
files under `skills/`.

Each skill is a fat markdown file with frontmatter (`triggers`,
`tools`, `writes_pages`), a routing entry in `skills/RESOLVER.md` (or
`AGENTS.md`), and a body that an LLM reads when the user's message
matches its triggers. Skills compose: `signal-detector` calls
`brain-ops`, `meeting-ingestion` calls `enrich`.

`skills/conventions/` holds cross-cutting rules every skill obeys:
brain-first lookup, filing rules, model routing, test-before-bulk.

## Consequences

- Releases stay focused. The v0.40 release notes describe new
  operations or schema, rarely new skills.
- Forks edit one markdown file and the change takes effect on next
  agent invocation. No `npm publish` for a routing tweak.
- The cost: skills are prompts. Their quality depends on LLM
  comprehension. The `gbrain skillify check` command + routing-eval
  fixtures catch regressions; the `check-resolvable` CI gate catches
  unreachable routing.

## Alternatives considered

- **Workflow engine (Temporal/Airflow-shaped).** Heavy; locks behavior
  into a typed DAG and forfeits the LLM's ability to compose tools
  freely.
- **All logic in code, skills as thin prompts.** What most agent
  frameworks do. Rejected because behavior changes then require a
  release, and the binary grows with every workflow.

## References

- `docs/ethos/THIN_HARNESS_FAT_SKILLS.md`
- `docs/ethos/MARKDOWN_SKILLS_AS_RECIPES.md`
- `skills/RESOLVER.md`
- `src/commands/skillify.ts`, `src/commands/skillpack-check.ts`
