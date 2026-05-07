# Stability roadmap — toward v1.0.0

> Status: solo-npm is in **v0.x experimental** but **code-side complete** as of v0.13.0. This document describes what **v1.0.0 will commit to** when the stability declaration ships. Until then, every minor release (`v0.X.0`) may break consumer wrappers — though we try not to.

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

**Code-level entry criteria: complete** as of v0.12.0. The hardening pass landed across v0.6.0 → v0.12.0; see [`CHANGELOG.md`](../CHANGELOG.md) for per-version detail.

What remains is **external/temporal** — these validate the API in the wild rather than adding code:

| Item | Why it's still open |
|---|---|
| Demo / showcase repo (`solo-npm-example`) | Not yet started. The maintainer's own portfolio (rfc-bcp47, ncbijs, npm-trust) currently serves as dogfood. |
| Wider adoption signal — at least one external user beyond the maintainer's portfolio | Depends on outreach. v1.0.0 won't declare stability without external validation. |
| Skill-spec drift caught once via the regression checklist before being noticed in production | Requires release cycles. |

**v1.0.0 ships when those three are met.** Internal patterns (phase numbering, AskUserQuestion option text, etc.) may keep iterating within v0.x even though the structural commitments above are already shaped to be v1.0.0-stable.

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

## Resolved questions (formerly "Open questions for v1.0.0")

Earlier drafts of this doc listed 4 questions to be answered before v1.0.0. Each is now resolved:

| Question | Resolution |
|---|---|
| **Trigger-phrase stability** | Phrases existing at v1.0.0 are stable (consumers may rely on them); see "Skill description trigger phrases that exist in v1.0.0" row in the commitments table. Adding new phrases in v1.x is free; removing a v1.0.0 phrase requires v2.0.0. |
| **Wrapper template stability** | Wrapper templates scaffolded by `/init` are **guidance, not stable API**. Users own their wrappers after `/init` runs; future `/init` invocations may produce different templates (this is non-breaking — the user's existing wrapper still works as long as the baseline `/solo-npm:<skill>` it invokes is stable). The commitment is the **scaffold artifact set's existence + location** (`.claude/skills/<skill>/SKILL.md` paths), not the exact wrapper body. |
| **`agent-skills` composition contract** | `agent-skills` is an **optional enhancement, not a dependency**. Skills detect at runtime if `agent-skills:debugging-and-error-recovery` is available; fall back to `AskUserQuestion` + manual flow otherwise (already implemented in `/hotfix` Phase D.2 and `/deps` Phase 4d as of v0.6.0). If addyosmani/agent-skills renames the skill, solo-npm gracefully degrades — not a v1.0.0 blocker. |
| **deps.dev API stability** | deps.dev v3 is considered stable as of v0.13.0. **Graceful degradation is already in place**: `/unpublish` Phase A.3 routes 5xx/timeout into the strict-stop branch (refuses unsafe operation); `/status` Phase 2 renders `(deps.dev unavailable)` and continues. If Google deprecates v3, solo-npm will ship a minor release with v4 support; the fallback already protects users in the interim. |

All 4 are **settled** for v1.0.0 — no further code or doc work required to resolve them.

## What's still outstanding for v1.0.0 (honest list)

Beyond the 3 external/temporal items in the punch list above, a small docs-staleness audit is the only other actual work item before v1.0.0:

| Item | Type | Owner |
|---|---|---|
| `CONTRIBUTING.md` audit — confirm it documents the v0.10.0–v0.13.0 patterns (H1–H8, Phase 0.5/0.5b, Phase −1, BCL convention, error-handling reference structure) | Docs | Maintainer |
| `README.md` audit — verify the "Commands at a glance" + detail subsections still reflect v0.13.0 surface | Docs | Maintainer |
| `docs/regression.md` audit — verify S1–S12 (early scenarios) haven't drifted relative to current skill bodies | Docs | Maintainer |
| Run the regression checklist (S1–S33) end-to-end at least once | Validation | Maintainer |
| `solo-npm-example` demo repo creation | New repo | Maintainer |
| Wider adoption signal | External | Outreach |
| Skill-spec drift caught once via the regression checklist before being noticed in production | Temporal | Time + release cycles |

The first 4 are within the maintainer's reach to do directly. The last 3 need the world.

**No new code is required for v1.0.0.** Everything from here is validation, audit, and outreach.
