# Outreach templates

> **Maintainer-only artifact**. This file is reusable templates for v1.0.0 announcement / community outreach. Not consumer-facing. Edit-as-you-go; the templates are starting points, not final copy.

## Elevator pitch (~30 seconds, for DMs / Discord / hallway)

solo-npm is a Claude Code marketplace plugin: 13 AI skills covering every npm command a solo developer actually runs — `publish`, `version`, `dist-tag`, `deprecate`, `audit`, `deps`, `owner`, `trust`, `unpublish` — plus quality gates (`verify`) and lifecycle helpers (`init`, `release`, `prerelease`, `hotfix`, `status`).

Each skill is a markdown body Claude reads at invocation. They share a strict-safety convention: H1–H8 error-handling patterns (OTP/2FA detection, registry rate-limit backoff, concurrent-invocation locking, etc.) plus universal pre-flight checks (detached HEAD, worktree, gh availability, submodule state). The result: when something goes wrong, you get a specific remediation block, not a verbatim `npm` error dump.

Built for solo devs who publish to npm and want OIDC Trusted Publishing + provenance attestations baked in, without writing `release.yml` from scratch. Currently dogfooded across rfc-bcp47, ncbijs, and npm-trust.

---

## Show HN — "Show HN: solo-npm — AI skills for every npm command a solo dev runs"

**Title**: `Show HN: solo-npm – AI skills for every npm command a solo dev runs`

**Body**:

```
Hey HN — I built a Claude Code marketplace plugin called solo-npm that turns
"how do I configure OIDC Trusted Publishing?" or "how do I unpublish a typo'd
package name?" into one-line prompts.

13 skills covering the npm CLI surface a solo dev actually uses:
- /solo-npm:init     — bootstrap release.yml + publishConfig + OIDC trust
- /solo-npm:release  — universal release entry; auto-chains pre-release lifecycle
- /solo-npm:hotfix   — backward maintenance with cherry-pick automation
- /solo-npm:audit    — security advisory triage + chain to dep upgrade
- /solo-npm:unpublish — wrong-name + rename-after-publish recovery
- + verify, prerelease, status, deps, dist-tag, deprecate, owner, trust

What's interesting (to me) is the strict-safety convention. After about 14
minor releases, every skill has the same error-handling pattern catalog (H1–H8):
OTP/2FA detection that surfaces a manual handoff instead of hanging on stdin;
3-attempt × 5s registry-propagation retry; per-package concurrent-invocation
file locks with stale-PID auto-cleanup; chain-target failure recovery that
captures child-skill STOPs and surfaces them in the parent context.

Not yet v1.0.0 (waiting on external adoption signal). Code-side complete; the
pre-v1.0.0 punch list at docs/stability.md is down to "run the 33-scenario
regression checklist end-to-end at least once" + outreach.

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

---

## r/node post (more casual)

**Title**: `I built solo-npm — a Claude Code plugin for the npm CLI surface`

**Body**:

```
After publishing rfc-bcp47, ncbijs, and npm-trust over the past year, I noticed
I was Claude-ifying the same npm flows over and over: "configure OIDC trust",
"audit and fix Tier-1 CVEs", "promote my beta to stable and clean up @next".

So I bundled the patterns into a Claude Code marketplace plugin: solo-npm.

13 skills, each a markdown body Claude reads at invocation:

- `/solo-npm:init`       — scaffolds release.yml + publishConfig + OIDC trust
- `/solo-npm:release`    — universal release entry; tag-triggered CI publish
- `/solo-npm:audit`      — security triage with risk classification
- `/solo-npm:unpublish`  — wrong-name + rename-after-publish recovery
- + 9 more (verify, hotfix, prerelease, status, deps, dist-tag, deprecate, owner, trust)

What I like about this format vs MCP / vs CLI tools:

- Markdown bodies are inspectable — `cat .claude/commands/release.md` shows
  exactly what the agent will do
- No code, no tests in the traditional sense. The "test suite" is a 33-scenario
  regression checklist (docs/regression.md) the maintainer runs before each
  release.
- Strict-safety baseline: every destructive npm call has the same error-handling
  shape (OTP detection, registry rate-limit backoff, concurrent-invocation
  locks). When something goes wrong, you get a specific remediation block, not
  a verbatim error dump.

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
- Be ready for "why not just use semantic-release" / "why not just use changesets" — answer: those handle a subset; solo-npm orchestrates the full lifecycle including post-publish recovery (`/unpublish`, `/dist-tag repoint`, `/deprecate`)

---

## awesome-claude-code submission template

**Section to target**: probably under "Marketplace plugins" or "npm tooling"

**Format** (one-line, follow the list's existing convention):

```
- [solo-npm](https://github.com/gagle/solo-npm) — AI skills for every npm command a solo dev runs (publish, dist-tag, deprecate, audit, deps, owner, trust, unpublish) with verify gates and OIDC provenance baked in.
```

**PR description**:

```
Adding solo-npm — a Claude Code marketplace plugin for solo developers
publishing to npm. 13 skills covering the npm CLI surface, with OIDC
Trusted Publishing + provenance attestations baked into the scaffolded
release workflow.

Code-side complete (v0.13.0+); v0.x experimental until external adoption
signal lands.
```

---

## v1.0.0 announcement post (for the project's own GitHub Discussions / Release notes)

**Title**: `solo-npm v1.0.0: stability declaration`

**Body**:

```
solo-npm has shipped v1.0.0.

Background: solo-npm is a Claude Code marketplace plugin with 13 AI skills
for npm release lifecycle management — from `/solo-npm:init` (bootstrap
OIDC Trusted Publishing) through `/solo-npm:release` (tag-triggered CI
publish with provenance) to `/solo-npm:unpublish` (wrong-name recovery).

What v1.0.0 means:

- **Skill invocation names** (`/solo-npm:release`, etc.) are stable — won't
  change without a v2.0.0 bump.
- **Skill identity** (what each skill does at a high level) is stable.
- **AskUserQuestion options** and **auto-chain destinations** that exist
  at v1.0.0 are stable.
- **State.json schema sections** (`trust`, `audit`, `pkgCheck`, `depsdev`)
  are stable.
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

- v0.6.0–v0.9.0: 12 skills shipped + scaffold artifacts established
- v0.10.0: `/solo-npm:unpublish` (skill #13) + the H1–H6 cross-skill
  error-handling pattern catalog
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
