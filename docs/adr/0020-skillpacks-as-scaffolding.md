# ADR-0020: Skillpacks as scaffolding, not managed-block

**Status:** Accepted (supersedes v0.19–v0.35.1 managed-block model)
**Landed:** v0.36

## Context

Skills (ADR-0009) live as markdown files. Multiple installs share a
canonical set, and gbrain wants a way to ship updates. The v0.19
approach was *managed blocks*: each shipped skill carried sentinel
markers (`<!-- gbrain:skillpack:begin --> ... <!-- gbrain:skillpack:end -->`)
plus a `cumulative-slugs="..."` receipt and a lockfile. `gbrain
skillpack install` would overwrite or merge based on content hashes,
then `gbrain skillpack uninstall` could remove cleanly.

By v0.35.1 the managed-block machinery was ~600 LOC of merge-mode
logic and the user experience was still confusing. Two problems
surfaced repeatedly:

- The receipt + lockfile model didn't accommodate users who hand-edited
  triggers between updates. Merge conflicts were silent.
- Paired-source declarations (which host-repo files a skill expects to
  edit alongside its SKILL.md) lived in `openclaw.plugin.json`, which
  is the wrong abstraction layer.

## Decision

Skillpacks are **scaffolding**, not managed installs. New surface:

- `gbrain skillpack scaffold <pack>` — one-time additive copy via the
  shared `copyArtifacts` helper. Refuses to overwrite existing files;
  partial-state fills missing paired sources declared in the SKILL.md
  frontmatter `sources:` array.
- `gbrain skillpack reference <pack>` — read-only diff lens between
  the shipped skillpack and the user's tree, with
  `--apply-clean-hunks` two-way auto-apply via a pure-JS unified-diff
  parser/applier.
- `gbrain skillpack migrate-fence <skill>` — one-shot strip of the
  legacy fence sentinels. Cumulative-slugs receipts fall back to row
  parsing. User-owned routing rows survive verbatim.
- `gbrain skillpack harvest <skill>` — inverse direction (host → gbrain).
  Symlink-reject + canonical-path containment + default-on privacy
  linter (`harvest-private-patterns.txt` + built-in patterns for
  emails, Slack channels, the literal "Wintermute"). Rollback on
  privacy match.

`install` and `uninstall` removed (clean break, no alias). Both exit
non-zero with a hint pointing at the replacement command. ~600 LOC of
managed-block machinery deleted; ~400 LOC of new modules + ~1000 LOC
of new tests.

The v0.37.0 wave added the third-party ecosystem on top: registry,
manifest (`gbrain-skillpack-v1`), TOFU trust prompts, the
`SKILLPACK_RUBRIC_V1` 10-dimension doctor (5 core + 5 quality badges).

## Consequences

- The mental model is git-shaped: scaffold once, edit in place, see
  drift via `reference`. Familiar from `npm init`, `yarn create`,
  `create-next-app`.
- User-owned routing rows in `RESOLVER.md` / `AGENTS.md` are never
  blown away by an update.
- The cost: no automatic "pull latest" for skills. Updates require an
  explicit `reference --apply-clean-hunks` pass.
- Old installs see exit-non-zero on `install` / `uninstall`. The
  migrate-fence command is the one-time bridge.

## Alternatives considered

- **Keep managed blocks, fix edge cases.** Considered. Rejected
  because the underlying merge semantics (auto-overwrite based on
  hashes) are wrong for files users edit by hand.
- **Skillpacks as immutable Docker-like layers.** Rejected: skills are
  meant to be edited; a layered FS is the wrong abstraction.

## References

- `src/commands/skillpack.ts`
- `src/core/skillpack/{bundle,scaffold,reference,migrate-fence,scrub-legacy,harvest,harvest-lint,copy,apply-hunks,diff-text}.ts`
- `docs/guides/skillpacks-as-scaffolding.md`
- `docs/designs/SKILLPACK_REGISTRY_V1_SPEC.md`
