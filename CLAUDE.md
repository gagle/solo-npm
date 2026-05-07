# CLAUDE.md — solo-npm project conventions

This file orients Claude when operating **inside the solo-npm repo** (e.g., editing skill bodies, scaffolding, or contributor work). It complements `CONTRIBUTING.md` and points at the canonical references rather than duplicating them.

> **Note**: This is for working ON solo-npm. When you're working IN a consumer repo that USES solo-npm, the consumer's own `.claude/skills/<name>/SKILL.md` wrappers + the marketplace-installed `/solo-npm:*` skills apply — not this file.

## What solo-npm is

A Claude Code marketplace plugin that ships **13 AI skills** for npm release lifecycle management. Skills live as markdown bodies in `.claude/commands/<name>.md`; Claude reads them at invocation time. No code, no tests in the traditional sense — the skill bodies ARE the deliverable, and `docs/regression.md` is the manual test suite.

## When working in this repo

### Adding or modifying a skill

1. Read `CONTRIBUTING.md` end-to-end first — it has the full **Patterns + conventions (v0.10.0–v0.13.0)** section that documents the phase numbering convention, the H1–H8 error-handling pattern catalog, the npm CLI BCL convention, the state.json conventions, the lock file convention, and the AskUserQuestion conventions.
2. The canonical reference for universal patterns is `/solo-npm:unpublish` Phase −1 (the most mature skill body). New universal patterns go there first; siblings reference by name.
3. Skill-specific patterns are inline in that skill's body — no need to centralize.
4. Preserve the frontmatter contract: `description: <one-liner under ~200 chars>` is required; `name:` is optional.

### When you need to know "what does pattern X mean"

| If you're looking for… | Read this |
|---|---|
| Phase numbering rules (`−1` / `0` / `0.5` / `0.5b` / `A`–`G`) | `CONTRIBUTING.md` §"Phase numbering" + `/solo-npm:unpublish` Phase 0 / Phase 0.5 / Phase 0.5b |
| H1–H8 error-handling patterns (OTP, cache corruption, auth-race, propagation lag, lock, chain-failure, tool reliability, rate-limit) | `CONTRIBUTING.md` §"H1–H8 error-handling pattern catalog" + `/solo-npm:unpublish` Phase −1.x for canonical implementations |
| npm CLI cross-version variations (BCL table) | `/solo-npm:unpublish` Phase −1.4b (the comprehensive ~30-pattern table) |
| state.json read-modify-write + atomic writes | `CONTRIBUTING.md` §"State.json conventions" + `/solo-npm:unpublish` Phase −1.4 |
| Lock file convention | `CONTRIBUTING.md` §"Lock file convention" + `/solo-npm:unpublish` Phase −1.8 |
| AskUserQuestion conventions | `CONTRIBUTING.md` §"AskUserQuestion conventions" |

### When you commit

- **Conventional commits** required (`feat:`, `fix:`, `chore:`, `docs:`, etc.).
- The commit-message DX is enforced by `commitlint` if present; otherwise just follow conventional commits by hand.
- Per the user's `~/.claude/CLAUDE.md` global instructions: prefer creating new commits over amending; never `--no-verify` unless explicitly requested.

### When you bump version

Per `docs/stability.md`:
- **Minor bump (`v0.X.0`)** for new skills, behavioral changes, API-shape changes, or when the user explicitly asks for visibility on a polish release.
- **Patch bump (`v0.X.Y`)** for doc-only fixes, cosmetic tweaks, bug fixes that don't change behavior.
- v1.0.0 won't ship until the 3 external/temporal items in `docs/stability.md` punch list are met. Until then, stay in v0.x.

### When you ship a release

1. Update `package.json` + `.claude-plugin/plugin.json` versions in lockstep.
2. Add a `CHANGELOG.md` entry under the new version heading.
3. Update `docs/stability.md` if the release moves the punch list.
4. Single commit, annotated tag (`git tag -a v0.X.Y -m "v0.X.Y"`), push both `main` and the tag.
5. SSL transients on `git push` to github.com are common — the `/release` skill body itself documents the 4-option remediation; for solo-npm's own pushes, just retry.

## What NOT to do in this repo

- **Don't enumerate skill counts in user-facing copy** (`README.md`, banner, plugin/marketplace descriptions). Use list-form ("publish, version, …") not count-form ("13 skills"). De-hardcoded since v0.8.0.
- **Don't duplicate canonical pattern prose** in sibling skills. Reference by name (e.g., "see `/unpublish` Phase −1.4b for the BCL table").
- **Don't add code** — solo-npm has no code, only markdown skill bodies + scaffolding templates. If something feels like it needs code, the answer is to write a clearer skill body.
- **Don't over-engineer fallbacks** — the H1–H8 patterns are the established safety net. New error-handling patterns should slot in as H9, H10, etc., with the same canonical-in-`/unpublish` + reference-from-siblings shape.

## Dogfooding

Per `CONTRIBUTING.md` §"Dogfooding": run `claude --plugin-dir .` from the repo root to load skills as plugin skills with the `solo-npm:` namespace. Edits to `.claude/commands/<name>.md` apply on next `/reload-plugins`.
