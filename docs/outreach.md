# Outreach templates

> **Maintainer-only artifact**. This file is reusable templates for v1.0.0 announcement / community outreach. Not consumer-facing. Edit-as-you-go; the templates are starting points, not final copy.

## Elevator pitch (~30 seconds, for DMs / Discord / hallway)

solo-npm is a Claude Code marketplace plugin: the AI release wizard for TypeScript npm packages. As of v0.19.0 the scope is narrowed to one mission — orchestrate everything a solo dev does around `npm publish`, from `/solo-npm:init` (bootstrap `release.yml` + OIDC Trusted Publishing) through `/solo-npm:release` (tag-triggered CI publish with provenance) and post-publish recovery (`/solo-npm:unpublish`, `/solo-npm:dist-tag`, `/solo-npm:deprecate`, `/solo-npm:hotfix`).

The differentiator is `/solo-npm:public-api`: it extracts the public API surface from `dist/`, diffs against the last released version, classifies the diff as patch/minor/major, and **HARD-STOPs `/release` Phase A.2b** if the requested bump is smaller than the diff requires. That single gate catches accidental breaking changes *before* the tag is pushed — the bug class no other tool in this space catches.

Each skill is a markdown body Claude reads at invocation. They share a strict-safety convention: H1–H8 error-handling patterns (OTP/2FA detection, registry rate-limit backoff, concurrent-invocation locking, SIGINT cleanup) plus universal pre-flight checks (detached HEAD, worktree, submodule state, `gh` availability). When something goes wrong, you get a specific remediation block, not a verbatim `npm` error dump.

Currently dogfooded across rfc-bcp47, ncbijs, and npm-trust.

---

## Show HN — "Show HN: solo-npm — the AI release wizard for TypeScript npm packages"

**Title**: `Show HN: solo-npm – the AI release wizard for TypeScript npm packages`

**Body**:

```
Hey HN — I built a Claude Code marketplace plugin called solo-npm that turns
"how do I configure OIDC Trusted Publishing?" or "did I just ship a breaking
change as a patch?" into one-line prompts.

As of v0.19.0 the scope is narrowed: solo-npm is the AI release wizard for
TypeScript npm packages. Everything orbits `npm publish` — from bootstrap
to post-publish recovery.

Headline skills:

- /solo-npm:init           — bootstrap release.yml + publishConfig + OIDC trust
- /solo-npm:release        — universal release entry; auto-chains pre-release
                             lifecycle; --changed-only for monorepos
- /solo-npm:public-api     — diffs your public API surface against the last
                             release; HARD-STOPs /release if the requested
                             version bump is too small for the actual diff
- /solo-npm:verify         — lint + typecheck + test + build + publint +
                             arethetypeswrong (attw) + exports orphan check +
                             tarball smoke-test
- /solo-npm:doctor         — 5-domain publish-health probe (trust, provenance,
                             config, security-gate, publish-readiness);
                             --fix chains into the relevant remediation skill
- /solo-npm:unpublish      — wrong-name + rename-after-publish recovery
- /solo-npm:hotfix         — patch a previous stable major on its <major>.x
                             branch with the correct dist-tag automatically
- /solo-npm:supply-chain   — provenance / license / age / typosquat audit
                             beyond CVE-only

Plus /solo-npm:audit, /solo-npm:deps, /solo-npm:dist-tag, /solo-npm:deprecate,
/solo-npm:owner, /solo-npm:trust, /solo-npm:prerelease, /solo-npm:status,
/solo-npm:secrets-audit, /solo-npm:lockfile-audit, /solo-npm:bundle-analyze,
/solo-npm:workspace, /solo-npm:migrate, /solo-npm:opt-in, /solo-npm:explain,
/solo-npm:toolchain, /solo-npm:smoke-test, /solo-npm:exports-check,
/solo-npm:types, /solo-npm:provenance-verify.

What's interesting (to me) is the strict-safety convention. Every skill
applies the same error-handling pattern catalog (H1–H8): OTP/2FA detection
that surfaces a manual handoff instead of hanging on stdin; 3-attempt × 5s
registry-propagation retry; per-package concurrent-invocation file locks
with stale-PID auto-cleanup; chain-target failure recovery that captures
child-skill STOPs and surfaces them in the parent context; SIGINT cleanup
that rolls back a local-only tag if you Ctrl+C between `git tag` and
`git push --tags`.

Not yet v1.0.0 (waiting on external adoption signal). Code-side complete;
the pre-v1.0.0 punch list at docs/stability.md is down to a single
end-to-end regression run + outreach.

Marketplace install:
  https://github.com/gagle/solo-npm

Curious whether anyone else has built tooling at the Claude-skill layer
(vs MCP or vs CLI). The marketplace plugin model is genuinely fun to build
in — markdown bodies that Claude reads, no code to compile.
```

**Notes for posting**:
- Best time: weekday morning US time
- Avoid: weekends, holidays, major news cycles
- The "Curious whether anyone else has built tooling at the Claude-skill layer" question is the discussion hook — gets people in adjacent spaces to comment
- Don't link directly to commercial services; HN dislikes that
- Be ready for "why not just use semantic-release / changesets / release-please?" — answer: those handle the version-bump + changelog subset. solo-npm orchestrates the full lifecycle including pre-publish gates that catch accidental breaking changes (`/public-api`), tarball-smoke-test before publish (`/smoke-test`), and post-publish recovery (`/unpublish`, `/dist-tag repoint`, `/deprecate`, `/hotfix`)

---

## r/node post (more casual)

**Title**: `I built solo-npm — a Claude Code plugin that wraps the whole npm publish/release lifecycle`

**Body**:

```
After publishing rfc-bcp47, ncbijs, and npm-trust over the past year, I noticed
I was Claude-ifying the same npm flows over and over: "configure OIDC trust",
"audit and fix Tier-1 CVEs", "promote my beta to stable and clean up @next",
"did I just ship a breaking change as a patch bump".

So I bundled the patterns into a Claude Code marketplace plugin: solo-npm.

The scope (as of v0.19.0) is narrowed to one mission: the AI release wizard
for TypeScript npm packages. Each skill is a markdown body Claude reads at
invocation.

Headline skills:

- `/solo-npm:init`        — scaffolds release.yml + publishConfig + OIDC trust
- `/solo-npm:release`     — universal release entry; tag-triggered CI publish
                            with SLSA provenance; --changed-only for monorepos
- `/solo-npm:public-api`  — diffs your public API surface vs last release;
                            HARD-STOPs the release if your version bump is
                            too small (e.g., patch on a breaking change)
- `/solo-npm:verify`      — lint + typecheck + test + build + publint + attw
                            + exports orphan check + tarball smoke-test
- `/solo-npm:doctor`      — 5-domain publish-health probe; --fix auto-remediates
- `/solo-npm:hotfix`      — backward maintenance on `<major>.x` branches
- `/solo-npm:unpublish`   — wrong-name + rename-after-publish recovery
- + a bunch more (verify gates, security audits, portfolio ops, monorepo
  workspace mgmt, CI-failure explanation, …)

What I like about this format vs MCP / vs CLI tools:

- Markdown bodies are inspectable — `cat .claude/commands/release.md` shows
  exactly what the agent will do
- No code, no tests in the traditional sense. The "test suite" is a
  regression checklist (docs/regression.md) the maintainer runs before each
  release
- Strict-safety baseline: every destructive npm call has the same error-handling
  shape (OTP detection, registry rate-limit backoff, concurrent-invocation
  locks). When something goes wrong, you get a specific remediation block, not
  a verbatim error dump

Install:
  /plugin marketplace add gagle/solo-npm
  /plugin install solo-npm@gllamas-skills

Or follow the README:
  https://github.com/gagle/solo-npm

Built with Claude Code itself, end-to-end. Pull requests are disabled — file
issues or discussions if you want to contribute. The README explains why.

Currently at v0.x experimental. v1.0.0 ships when external adoption + a few
validation signals land.
```

**Notes for posting**:
- r/node is more practical than HN — emphasize the working-code aspect
- The "no code, no tests" framing is unusual; lean into it
- Be ready for "why not just use semantic-release / changesets" — answer: those handle the version-bump + changelog subset. solo-npm orchestrates the full lifecycle including pre-publish gates (`/public-api`, `/types`, `/smoke-test`) and post-publish recovery (`/unpublish`, `/dist-tag repoint`, `/deprecate`, `/hotfix`)

---

## awesome-claude-code submission template

**Section to target**: probably under "Marketplace plugins" or "npm tooling"

**Format** (one-line, follow the list's existing convention):

```
- [solo-npm](https://github.com/gagle/solo-npm) — AI release wizard for TypeScript npm packages: bootstrap OIDC + provenance, run pre-publish gates (publint, attw, exports-check, smoke-test, public-API diff), tag-triggered release, post-publish recovery (unpublish, dist-tag, deprecate, hotfix).
```

**PR description**:

```
Adding solo-npm — a Claude Code marketplace plugin: the AI release wizard
for TypeScript npm packages. Bootstrap (OIDC Trusted Publishing +
provenance), pre-publish gates (publint + arethetypeswrong + exports
orphan check + tarball smoke-test + public-API surface diff), tag-triggered
release CI, post-publish recovery, monorepo --changed-only mode.

Code-side complete (v0.19.0); v0.x experimental until external adoption
signal lands.
```

---

## v1.0.0 announcement post (for the project's own GitHub Discussions / Release notes)

**Title**: `solo-npm v1.0.0: stability declaration`

**Body**:

```
solo-npm has shipped v1.0.0.

Background: solo-npm is a Claude Code marketplace plugin — the AI release
wizard for TypeScript npm packages. From `/solo-npm:init` (bootstrap OIDC
Trusted Publishing) through `/solo-npm:release` (tag-triggered CI publish
with provenance) to `/solo-npm:unpublish` (wrong-name recovery) and
`/solo-npm:hotfix` (backward maintenance on legacy majors).

What v1.0.0 means:

- **Skill invocation names** (`/solo-npm:release`, etc.) are stable — won't
  change without a v2.0.0 bump.
- **Skill identity** (what each skill does at a high level) is stable.
- **AskUserQuestion options** and **auto-chain destinations** that exist
  at v1.0.0 are stable.
- **State.json schema sections** (`trust`, `audit`, `pkgCheck`, `depsdev`,
  `toolchain`) are stable.
- **`/init` scaffold artifact set** is stable.
- **The conventions documented in CONTRIBUTING.md** (phase numbering,
  H1–H8 error-handling pattern catalog, npm CLI BCL convention,
  state.json conventions, lock file convention, AskUserQuestion shape)
  are stable.

Breaking any of the above requires v2.0.0 (rare). Adding new skills, new
phases, new options, new conventions — all free in v1.x.

What's NOT promised even after v1.0.0:

- Exact wording in skill bodies (Claude reads markdown; we'll improve clarity)
- Exact phase numbering (Phase A.3 might split into A.3.1 + A.3.2)
- The order of AskUserQuestion options (the existence is stable; ordering
  may evolve for UX)
- Internal cache schemas beyond their declared sections

See [docs/stability.md] for the full commitment table.

What changed during v0.x:

- v0.6.0–v0.9.0: initial skill set + scaffold artifacts established
- v0.10.0: `/solo-npm:unpublish` + the H1–H6 cross-skill error-handling
  pattern catalog
- v0.10.1: Tier-1 strict-safety hardening (universal Phase −1, tag-collision
  pre-flight, git push categorization)
- v0.11.0: Tier-2 hardening (concrete H3/H4/H5/H6 implementations,
  schema resilience, AI-prompt regex validation)
- v0.12.0: Tier-3 stability (rate-limit backoff, SSL/TLS remediation,
  SIGINT cleanup, npm CLI BCL extensions)
- v0.13.0: Tier-4 polish (proactive rate-limit tracker, comprehensive npm
  BCL table, gh GraphQL fan-in, CRLF-in-tarball detection)
- v0.14.0: docs alignment pass (CONTRIBUTING.md + README.md + regression
  scenarios)
- v0.15.0: 6-item polish consolidation (description alignment, prompts.md
  refresh, CLAUDE.md, outreach material)
- v0.16.0: Phase −0 help mode + npm-trust pin bump + TOCs + CHANGELOG
  condensation + manifest CI
- v0.17.0: cross-tool integration overhaul (npm-trust ^0.11 +
  prepare-dist ^1.1)
- v0.18.0: dropped npm-package version pins; track `@latest` via runtime
  capability probe
- v0.19.0: PART III narrowing — 16 new skills landed: `/doctor`,
  `/toolchain`, `/types`, `/public-api`, `/exports-check`, `/smoke-test`,
  `/provenance-verify`, `/supply-chain`, `/lockfile-audit`,
  `/secrets-audit`, `/workspace`, `/bundle-analyze`, `/migrate`,
  `/opt-in`, `/explain`, plus `/release --changed-only`
- v0.19.1: docs alignment patch (no behavioral changes)

Thanks to early adopters who dogfooded across rfc-bcp47, ncbijs, and
npm-trust during the v0.x experimental period.

Install: https://github.com/gagle/solo-npm
```

**Notes for posting**:
- Edit the version-list to reflect the actual final state when v1.0.0 ships
- The "early adopters" line should mention any external users by handle if they consented
- Consider also a short Twitter/Mastodon post pointing at this announcement

---

## When to publish each template

| Template | When |
|---|---|
| Elevator pitch | DMs, Discord, conference hallway, Slack — anytime |
| Show HN | At v1.0.0 declaration; weekday morning US time |
| r/node post | Same window as HN, or 1 day later |
| awesome-claude-code PR | At v1.0.0 declaration |
| v1.0.0 announcement | The morning v1.0.0 ships; pin in GitHub Discussions |

## Maintenance

When the project state changes substantially (new headline skill, major architectural shift), update these templates so they're ready when needed. They're cheaper to keep current than to write under deadline pressure at v1.0.0 time.
