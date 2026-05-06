# Stability roadmap — toward v1.0.0

> Status: solo-npm is in **v0.x experimental**. This document describes what **v1.0.0 will commit to** when the stability declaration ships. Until then, every minor release (`v0.X.0`) may break consumer wrappers — though we try not to.

## Why this doc exists

solo-npm is consumed by other repos (rfc-bcp47, ncbijs, future adopters) via thin wrappers at `.claude/skills/release/SKILL.md`. Those wrappers invoke `/solo-npm:release` and expect specific behavior. As we evolve the skill set, consumers need to know:

- What's safe to depend on?
- What might break in the next minor bump?
- When will the API freeze?

This roadmap answers all three.

## Stability levels (current state)

| State | Skills | Stability promise |
|---|---|---|
| **Stable** | Most user-facing skill names + their primary purpose | Names won't change; core behavior won't disappear |
| **Active iteration** | Phase numbering, `AskUserQuestion` option text, auto-chain conditions, internal cache schema | May change between minor versions |
| **Pre-1.0 latitude** | Anything else | Free to re-architect if it improves the design |

While in v0.x, we still try to avoid gratuitous breakage — but we reserve the right to fix things properly even if it requires a minor bump.

## What v1.0.0 will commit to

When we ship v1.0.0, the following become **stable** and require a v2.0.0 bump to break:

| Stable in v1.0.0 | Free to evolve in v1.x |
|---|---|
| Skill **invocation names** (`/solo-npm:release`, etc.) | Spec wording / phrase improvements (Claude reads the new wording) |
| Skill **identity** — what each skill does at a high level | New phases appended at the end of an existing skill |
| **AskUserQuestion options** that exist in v1.0.0 | New options added to existing `AskUserQuestion` gates |
| **Auto-chain destinations** that exist in v1.0.0 | New auto-chains added |
| `.solo-npm/state.json` schema **sections that exist** in v1.0.0 (`trust`, `audit`, `pkgCheck`, `depsdev`) | New state.json sections added |
| **Skill description trigger phrases** that exist in v1.0.0 | New trigger phrases added |
| `/init` **scaffold artifact set** (`release.yml` shape, `publishConfig` fields, wrapper templates) | Internal phase organization changes (e.g., Phase 1c → 1d numbering) |
| External tool dependencies (publint pinned major; `npm-trust` required; `gh` optional) | Bump pinned tool minor versions |
| The **convention** documented in CONTRIBUTING.md (phase naming, AskUserQuestion shape, etc.) | Add new conventions for new patterns |

What requires **v2.0.0**:

- Renaming or removing a shipped skill
- Removing an `AskUserQuestion` option that existed in v1.0.0
- Removing an auto-chain that existed in v1.0.0
- Breaking the `.solo-npm/state.json` schema (e.g., renaming a field, changing a value type)
- Removing a scaffold artifact `/init` produces
- Changing the default behavior of a skill in a way that surprises existing consumers

## What's NOT promised, even after v1.0.0

These are explicit non-promises so consumers don't accidentally depend on them:

- **Exact wording** in skill bodies. Claude reads the markdown; we'll improve clarity over time.
- **Exact phase numbering** (e.g., `/release` Phase A.3 might become A.3.1 + A.3.2 if we split it).
- **The order of `AskUserQuestion` options** (we may reorder for better UX; the *existence* is stable).
- **Internal scratch state** (e.g., temporary fields in skill execution that aren't part of `state.json`).
- **External tool versions** beyond their pinned major (we may bump publint 0.3.x → 0.3.y).
- **Output formatting** (e.g., the exact format of `/status` table rows).
- **Performance** (we may rework cache strategies; behavior stays the same).

## What needs to land before v1.0.0

The pre-v1.0.0 punch list (status as of v0.9.0):

| Item | Status | Where |
|---|---|---|
| De-hardcode skill counts in user-facing copy | ✓ shipped v0.8.0 | README, CONTRIBUTING, lifecycle.svg, npm-coverage.md |
| Fix the 3 doc drift items found in audit | ✓ shipped v0.8.0 | README line 154, CONTRIBUTING line 92, prompts.md "Day 1" scenario |
| `/release` PACKAGE_NOT_PUBLISHED auto-chain to `/init` | ✓ shipped v0.8.0 | release.md Phase A.3 |
| Phase naming convention codified | ✓ shipped v0.8.0 | CONTRIBUTING.md |
| Cache `lastSize` retention policy | ✓ shipped v0.8.0 | release.md C.7.6, prerelease.md C.7.6, hotfix.md E.7 |
| Clarify `/audit` "Fix Tier 1" wording | ✓ shipped v0.9.0 | audit.md Phase 5 |
| `docs/stability.md` (this doc) | ✓ shipped v0.9.0 | docs/stability.md |
| `docs/regression.md` (manual scenario walkthrough) | ✓ shipped v0.9.0 | docs/regression.md |
| `deps.dev` integration in `/status` | ✓ shipped v0.9.0 | status.md Phase 2 + Phase 3 |
| **Demo / showcase repo** (`solo-npm-example`) | open | not yet started |
| **Wider adoption signal** — at least one external user beyond rfc-bcp47/ncbijs | open | depends on outreach |
| **Skill-spec drift caught at least once via the regression checklist** before being noticed in production | open | requires release cycles |

The first 9 items are landed. The remaining 3 are external/temporal — they need real-world usage to validate that the API holds up. **v1.0.0 won't ship until those are met.**

## Versioning policy in v0.x

Until v1.0.0, we use **minor bumps** for any meaningful change:

- `v0.X.0` for new skills, behavioral changes, or API-shape changes
- `v0.X.Y` (patch) for doc-only fixes, cosmetic tweaks, bug fixes that don't change behavior

Migrating between minor versions in v0.x may require updating consumer wrappers. We document migration steps in each `CHANGELOG.md` entry under "Migration".

After v1.0.0 we switch to standard semver:
- `v1.X.0` for additive features
- `v1.X.Y` for fixes
- `v2.0.0` for breaking changes (rare; reserved for the kinds of breakage listed above)

## How to depend on solo-npm safely (today)

If you're consuming solo-npm via the marketplace plugin in your own repo:

1. **Pin the plugin version** in `.claude/settings.json` if you want stability over latest:
   - Default behavior tracks the marketplace's main; you get every minor release
   - Pin syntax (per Claude Code marketplace docs) lets you stay on a known version
2. **Wrap behavior, not implementation**: your `.claude/skills/release/SKILL.md` should describe *repo-specific narrative* (workspace shape, verify commands), not re-implement what the baseline does. If the baseline changes, your wrapper still works.
3. **Track this doc**: when v1.0.0 ships, this doc updates to "stable". Until then, expect minor-version churn.

## Open questions for v1.0.0

These are deliberately unresolved; will be answered before v1.0.0 ships:

- **Stable trigger-phrase set**: how many trigger phrases per skill is canonical? Is a phrase added in v1.X.Y a stability promise or just a quality-of-life addition?
- **Wrapper template stability**: `/init` scaffolds wrapper templates. Are those wrapper bodies stable, or can we improve them?
- **`agent-skills` composition contract**: we name `agent-skills:debugging-and-error-recovery` in `/hotfix` Phase D.2 and `/deps` Phase 4d. If `agent-skills` renames that skill, we'd silently break. Should we pin the dependency?
- **deps.dev API stability**: Google maintains the API; if they deprecate v3, we need a fallback. Document the fallback path before v1.0.0.

These will be answered in `docs/stability.md` when v1.0.0 ships. For now, they're known-uncertain.
