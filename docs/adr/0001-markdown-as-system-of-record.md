# ADR-0001: Markdown as system of record

**Status:** Accepted
**Landed:** v0.1 (foundational); formalized by `docs/architecture/system-of-record.md` and the `check-system-of-record.sh` CI gate

## Context

GBrain stores personal knowledge that takes years to accumulate. Two natural
places to put it: (a) a database, where search is fast and joins are easy, or
(b) plain markdown files in a git repo, where the user owns the bytes and can
read them with any editor.

A DB-first design would have made retrieval simpler and removed the import
pipeline, but it would also have made the user a tenant of their own brain.
Loss of the DB would mean loss of the knowledge. Multi-machine sync would need
a custom protocol. Privacy review would mean SQL forensics.

## Decision

The markdown files (plus YAML frontmatter) are the canonical state. The
Postgres or PGLite database is a derived cache: chunks, embeddings, the graph,
the takes table — all rebuildable from the markdown by running
`gbrain sync && gbrain extract all`.

The user never has to back up the DB. They back up their brain repo (git push)
and the DB regenerates on the next sync.

## Consequences

- Disaster recovery is one command. `gbrain rebuild --confirm-destructive`
  wipes the DB and re-imports from the repo.
- Multi-machine sync is `git push` / `git pull`. The second machine's DB
  rebuilds locally on next sync.
- Sensitive content is grep-able and removable as files, not SQL rows.
- The cost: import pipelines must be idempotent and order-tolerant, since
  any file may be re-imported at any time. Every write path keys on
  `(slug, content_hash)` to skip no-op rewrites.

## Alternatives considered

- **DB as primary, markdown as export.** Rejected because export-on-demand is
  always lossy and never tested in the hot path.
- **Hybrid (some content DB-only, some markdown).** Rejected because the
  invariant "everything you care about is in the repo" is what makes recovery
  one command.

## References

- `docs/architecture/system-of-record.md`
- `scripts/check-system-of-record.sh` (CI gate against write paths that would
  introduce DB-only state)
- `src/commands/rebuild.ts`
