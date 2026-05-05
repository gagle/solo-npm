# Prompts → solo-npm commands

Real-world prompts a solo dev (or AI agent) might type, and which solo-npm command they trigger. Use this as a quick reference for what the skills are listening for, and as inspiration for how minimal a prompt can be while still triggering the right flow.

## Per-command prompts

### `/solo-npm:init` — bootstrap a fresh repo

Scenario: just `git init`'d a new package and wrote some code.

| Prompt | What happens |
|---|---|
| *"Integrate solo-npm into this repo"* | Canonical first prompt. Claude reads README, writes `.claude/settings.json`, prompts marketplace + plugin install; once installed, runs `/solo-npm:init`. |
| *"Set this up for tag-triggered npm publishing"* | Direct match to init's description. Scaffolds release.yml with OIDC + provenance. |
| *"Add the OIDC release workflow"* | Init scaffolds release.yml + publishConfig + .nvmrc + consumer wrappers. |
| *"I just `npm init`'d this — wire it up for solo-npm releases"* | Claude reads context (just-created package.json, no release.yml), invokes `/solo-npm:init`. |

### `/solo-npm:trust` — configure OIDC trust

Scenario: first publish landed, ready for Trusted Publishing. Or new package added to a monorepo.

| Prompt | What happens |
|---|---|
| *"Configure OIDC trust for these packages"* | Direct match. Walks user through web 2FA. |
| *"Set up Trusted Publishing on npm"* | Same. |
| *"I just published @scope/foo for the first time, configure trust"* | Trust skill detects `--only-new`. |
| *"Why is my CI failing on publish?"* (failure is missing trust) | Claude diagnoses, invokes trust. |
| *"Add trust config for @scope/newpackage"* | Trust skill targets just that package. |

### `/solo-npm:release` — universal release entry point

Scenario: time to ship. After v0.5.4, `/release` auto-routes to `/solo-npm:prerelease` if version is pre-release-shape, runs stable flow otherwise.

| Prompt | What happens |
|---|---|
| *"Ship it"* / *"Release this"* / *"Cut a release"* | Classic. Phase A → B → C. |
| *"Time to release v0.5.4"* | Auto-bump derives from commits; user clicks Proceed at v0.5.4. |
| *"Release"* | Bare command; same. |
| *"Bump the beta counter and ship"* (when on `1.6.0-beta.1`) | `/release` detects pre-release → auto-chains to `/solo-npm:prerelease` → user picks Bump. |
| *"I'm done, push it out"* | Matches stable release flow. |
| *"Get this on npm"* | Same. |

### `/solo-npm:verify` — quality gates

Scenario: mid-development, want to confirm nothing's broken.

| Prompt | What happens |
|---|---|
| *"Verify"* / *"Run the gates"* / *"Make sure everything passes"* | Lint + typecheck + test + build sequentially. |
| *"Did I break anything?"* | Matches description. |
| *"Run all the tests"* | Verify covers test as one of its steps. |
| *"Quick sanity check"* | Could match verify or status; verify wins (it's about confirming nothing's broken). |
| *"Did `pnpm test` pass?"* | Claude infers verify is the right wrapper. |

### `/solo-npm:status` — portfolio dashboard

Scenario: morning coffee, checking on packages.

| Prompt | What happens |
|---|---|
| *"How are my packages doing?"* | Reads cache + npm registry, renders dashboard. |
| *"Show me the portfolio dashboard"* | Direct match. |
| *"What's pending release?"* | Drift column shows unreleased commits. |
| *"Anything published recently?"* | Last-publish column. |
| *"How many downloads last week?"* | Downloads column. |
| *"Trust state across my packages"* | Reads `.solo-npm/state.json` cache. |
| *"What's open in GitHub for my packages?"* | gh repo issues column. |

### `/solo-npm:audit` — security audit with risk triage

Scenario: a CVE just hit Twitter, or it's monthly maintenance day.

| Prompt | What happens |
|---|---|
| *"Audit my packages for vulnerabilities"* | Runs pnpm audit, classifies tiers. |
| *"Are there any CVEs I need to fix?"* | Direct match. |
| *"Security audit"* / *"What's vulnerable?"* | Same. |
| *"I saw a CVE for tar-fs — am I affected?"* | Audit lists per-advisory, includes tar-fs if vulnerable. |
| *"Run a security check before release"* | Invoked pre-release as sanity. |
| *"Anything urgent in my deps?"* | Tier 1 entries surface first. |

### `/solo-npm:deps` — dep upgrade orchestrator

Scenario: monthly maintenance, or audit said "fix Tier 1 now".

| Prompt | What happens |
|---|---|
| *"Update my deps"* / *"Bump packages"* | Runs outdated, classifies, batches with /verify gates. |
| *"Refresh dependencies"* | Same. |
| *"Apply the CVE fixes from the audit"* | Chained from audit; deps targets cve-tier-1. |
| *"Upgrade typescript to v6"* | Major-bump path; surfaces release notes; user gates per-dep. |
| *"Catch me up on dep versions"* | Full classify + plan. |
| *"Don't break anything, but update what's safe"* | Trivial+safe path. |

### `/solo-npm:prerelease` — explicit pre-release entry

Scenario: starting a beta line, or transitioning beta → stable.

| Prompt | What happens |
|---|---|
| *"Start a beta for v2"* | Asks identifier (beta) + base bump (major); ships `2.0.0-beta.0` to `@next`. |
| *"Cut a release candidate"* | Same flow with `rc` identifier. |
| *"I want to start an alpha"* | Start-line path. |
| *"Promote the beta to stable"* (already on `1.6.0-beta.3`) | Direct invocation; user picks Promote → publishes `1.6.0` to `@latest`. (Also accessible via `/release` auto-chain.) |
| *"Ship a beta of these breaking changes"* | Start-line path with major bump. |
| *"Publish v2.0.0-beta.1"* | Start-line; user accepts the computed version. |
| *"I want a `next` channel publish for testing"* | Pre-release publishes to `@next` by default. |

### `/solo-npm:hotfix` — backward maintenance

Scenario: active users on v1.5 hit a bug; main is in v2 development.

| Prompt | What happens |
|---|---|
| *"There's a bug in v1.5 — fix it and ship"* | Detects v1 line, switches to `1.x` branch, asks for fix description, applies, releases v1.5.1. |
| *"Hotfix the v1 rate limiter — it crashes on 429"* | Branches; agent applies fix per description; ships. |
| *"I need to patch v1.5.0"* | Direct match. |
| *"Backport this fix to v1"* | Hotfix workflow; agent re-applies the fix on `1.x` branch. |
| *"v1 users are reporting a parsing bug — fix it"* | Hotfix workflow. |
| *"Patch the previous major"* | Hotfix. |

### `/solo-npm:dist-tag` — post-publish dist-tag management

Scenario: stale `@next` after promote, botched release rollback, or opt-in channel publish.

| Prompt | What happens |
|---|---|
| *"Cleanup stale @next across all packages"* | Detects packages where `@next` points at a superseded pre-release; bulk `npm dist-tag rm` after gate. |
| *"Repoint @latest to 1.5.2 — 1.6.0 has a bug"* | Repoints `@latest` (recovery before fix-forward). |
| *"Add @canary to 1.6.0-experimental.2 on all packages"* | Adds the `@canary` tag pointing at the experimental version. |
| *"Remove @next from @ncbijs/eutils"* | Single-package rm. |
| *"What dist-tags are set on my packages?"* | Read-only `ls` — no gate, just renders the table. |
| *"What dist-tags do my packages have?"* | Same. |

### `/solo-npm:deprecate` — retire versions cleanly

Scenario: post-major-release EOL, CVE response, bad-release recovery.

| Prompt | What happens |
|---|---|
| *"Deprecate all 1.x with message 'v1.x is EOL — migrate to v2'"* | Mass-deprecate every 1.x version across the portfolio with one gate. |
| *"Mark 1.6.0 as do-not-use because of a data bug"* | Single-version deprecation; agent derives "Do not use — data bug; upgrade to 1.6.1." |
| *"Deprecate <2.0.0 across all packages"* | Range-bounded deprecation across portfolio. |
| *"Undeprecate 1.5.0 of @ncbijs/eutils"* | Lifts the deprecation (sets message to `""`). |
| *"Vulnerable to CVE-XXXX — deprecate the affected versions"* | Range deprecation triggered from /audit chain or directly. |

### `/solo-npm:owner` — manage maintainers

Scenario: bus-factor mitigation, ownership transfer, audit.

| Prompt | What happens |
|---|---|
| *"Add @backup-maintainer to all my packages"* | Bulk owner add across portfolio after gate. |
| *"Show me who can publish each package"* | Read-only `ls` portfolio audit; bus-factor risk highlighted. |
| *"Audit ownership across portfolio"* | Same as above. |
| *"Remove @old-collaborator from @ncbijs/eutils"* | Single-package rm; rejects sole-owner removal. |
| *"Who has publish access to npm-trust?"* | Single-package owner list. |

## Compound real-world workflows

### Day 1: brand-new package → on npm with provenance

```
You:  Integrate solo-npm into this repo.
       (agent reads README, writes settings.json, prompts install)
You:  [accept install prompts]
You:  Now bootstrap.
       (agent runs /solo-npm:init → scaffolds workflow + wrappers + cache →
        Phase 2 detects PACKAGE_NOT_PUBLISHED → tells you to publish manually first)
You:  I just ran `npm publish --provenance=false --access public` — continue.
       (agent re-invokes init → Phase 2 passes → Phase 3 chains into /solo-npm:trust)
You:  [accept "Configure trust for 1 package?" prompt; do the npm web 2FA]
       (agent: trust complete; cache populated; init Phase 4 reports done)
You:  Now ship the first real release.
       (agent runs /release → Phase A passes → B.1 "First release: 0.1.0/1.0.0?" → user picks)
```

### Routine release day

```
You:  Ship a release.
       (agent runs /release → Phase A silent (cache hot path, zero npm calls) →
        B.5 "Release v0.5.4? Proceed/Abort" → user clicks Proceed →
        Phase C executes → tarball on npm with provenance)
```

### Monthly maintenance window

```
You:  How are my packages doing?
       (agent runs /solo-npm:status → renders dashboard → flags 2 stale provenance, 1 vulnerability)
You:  Audit my deps.
       (agent runs /solo-npm:audit → finds 1 critical+runtime+direct vuln in `tar-fs`, 4 lower)
You:  Fix Tier 1 now.
       (agent chains into /solo-npm:deps cve-tier-1 → upgrades tar-fs → /verify → commit)
You:  Ship the patch.
       (agent runs /release → patch bump → v0.5.5 to @latest)
```

### Major version transition (beta → stable)

```
You:  Start a beta for v2.
       (agent runs /solo-npm:prerelease → asks identifier ("beta") → asks base bump ("major") →
        plan: 1.5.0 → 2.0.0-beta.0 → user confirms → publishes to @next)

[a week of feature work and `feat:` commits later]

You:  Ship the next beta.
       (agent runs /release → version is 2.0.0-beta.0 (pre-release) →
        auto-chains to /solo-npm:prerelease →
        Bump/Promote/Abort → user picks Bump → publishes 2.0.0-beta.1 to @next)

[two more bumps, ready to promote]

You:  Promote v2 to stable.
       (agent runs /solo-npm:prerelease directly → state is pre-release →
        Bump/Promote/Abort → user picks Promote → publishes 2.0.0 to @latest →
        Phase E offers /solo-npm:dist-tag cleanup-stale → user picks Yes →
        @next removed from all affected packages)

You:  [agent surfaces /release Phase G post-major gate]
       "Deprecate v1.x now with message 'v1.x is EOL — migrate to v2.0.0+'?"
       Yes / Customize / Defer
       (user picks Yes → chains to /solo-npm:deprecate →
        deprecates all 1.x versions across the portfolio with the migration message)
```

### Hotfix to v1 while v2-beta is in flight

```
You:  Users on v1.5 are hitting a bug where the rate limiter doesn't honor
      Retry-After. Fix it on the v1 line and ship.
       (agent runs /solo-npm:hotfix →
        Phase A: detects candidate majors [1] → "Which major?" → user picks 1.x →
        Phase B: switches to 1.x branch from v1.5.0 →
        Phase C: target dist-tag = @latest (v1 still current stable, since v2 is pre-release) →
        Phase D: surfaces "describe the fix" → user already described in initial prompt →
        agent applies fix to src/rate-limiter.ts on 1.x branch → /verify passes →
        Phase E: "Release v1.5.1 to @latest?" → user clicks Proceed →
        commits, tags, pushes, CI publishes to @latest →
        Phase F: returns to main)
You:  Now ship the next v2 beta with this same fix.
       (Phase F.5 of /hotfix already offered "Forward-port to main"; user picked
        "Yes — cherry-pick and ship". Agent did `git cherry-pick <hotfix-sha>` on
        main, ran /verify, then chained into /release → version is pre-release-shape
        (2.0.0-beta.4) → auto-chained to /solo-npm:prerelease → user picked Bump →
        2.0.0-beta.5 published to @next.)
```

Forward-porting is automated since v0.6.0 via `/hotfix` Phase F.5. The cherry-pick
is gated by an `AskUserQuestion` (Yes ship/Yes cherry-pick only/No skip); conflicts
surface a structured handoff (resolve manually, then continue with /release).

### CVE landed in popular dep

```
You:  I saw the GHSA-xxxx CVE in tar-fs. Audit + fix.
       (agent runs /solo-npm:audit → confirms repo affected → tier 1 entry →
        chains into /solo-npm:deps cve-tier-1 → upgrades tar-fs → /verify → commit)
You:  Ship the patch to all affected packages.
       (agent runs /release → patch bump → v0.5.5 to @latest)
```

### CVE landed but upgrade is blocked

Scenario: CVE affects dep used in v1.x, but upgrading the dep requires a breaking change you can't accept yet. Deprecate the affected versions to warn consumers, ship the upgraded version on v2.x via a major bump.

```
You:  GHSA-yyyy hits @ncbijs/* v1.x — upgrade is blocked by Node compat.
You:  Audit it.
       (agent runs /solo-npm:audit → finds Tier 1 advisories →
        Phase 5 gate offers: Fix Tier 1 / Fix Tier 1+2 / Deprecate affected versions / Show details / Defer)
You:  Pick "Deprecate affected versions"
       (agent extracts version range from advisory's vulnerable_versions →
        chains to /solo-npm:deprecate with RANGE=<2.0.0 and MESSAGE pre-filled with CVE info →
        user reviews + Proceed → mass-deprecates 1.x versions →
        consumers running `npm i @ncbijs/eutils@^1` see the warning)
```

### Botched release recovery

Scenario: just shipped 1.6.0 with a bug. Need to repoint @latest to 1.5.2 immediately, then deprecate 1.6.0, then fix-forward.

```
You:  1.6.0 has a regression — repoint @latest to 1.5.2 across all packages.
       (agent runs /solo-npm:dist-tag → operation=repoint, version=1.5.2, scope=all →
        gate: "Repoint @latest from 1.6.0 → 1.5.2 on N packages?" → user Proceeds →
        all @latest repointed; consumers running `npm i pkg` get 1.5.2 again)

You:  Now mark 1.6.0 as do-not-use.
       (agent runs /solo-npm:deprecate → range=1.6.0, scope=all,
        message='Do not use — regression in <feature>; downgraded @latest to 1.5.2; fix coming in 1.6.1' →
        user Proceeds → 1.6.0 deprecated everywhere)

[fix lands]

You:  Ship 1.6.1.
       (agent runs /release → version 1.6.1 → @latest now points at 1.6.1)

You:  [agent could surface /solo-npm:deprecate undeprecate offer for 1.6.0 if user wants —
        otherwise the deprecation persists which is fine; 1.6.0 stays installed-but-warned]
```

### Bus-factor audit

Scenario: ahead of vacation, want to make sure someone else can publish if needed.

```
You:  Show me who can publish each package.
       (agent runs /solo-npm:owner → operation=ls, scope=all →
        renders portfolio table → highlights sole-owner packages as bus-factor risk)

You:  Add @backup-maintainer to all packages where I'm sole owner.
       (agent runs /solo-npm:owner → operation=add, user=backup-maintainer, scope=all →
        gate: "Add @backup-maintainer to N packages?" → user Proceeds → all packages now multi-owner)
```

### Resuming work in a previously-set-up repo (cross-machine)

```
You:  [open the repo in Claude Code on a new laptop]
       [Claude Code prompts: "install gllamas-skills marketplace? install solo-npm@gllamas-skills?"]
You:  [accept]
You:  How are my packages?
       (status reads cached state.json, renders dashboard with no fresh npm calls —
        cross-machine cache works because state.json is committed)
```

### GitHub Release page auto-fills on every release (v0.7.0)

```
You:  Ship a release.
       (agent runs /release → Phase A silent → B.5 Proceed → Phase C executes →
        C.7 verifies registry attestation → C.7.5 creates GitHub Release with
        the just-prepended CHANGELOG entry as the body → C.7.6 caches
        unpackedSize for next /verify Tier 4 baseline → C.8 final notification)

[anyone watching the gagle/solo-npm repo gets a GitHub notification with the
 release notes, not just a "tag pushed" event]
```

For pre-releases: GitHub Release is marked `--prerelease` so it shows the badge.
For hotfixes on legacy lines: `--latest=false` so the GitHub Releases UI keeps
the latest-badge on the current main-line stable.

If `gh` is not installed or not authenticated: skipped silently with a non-fatal
warning. The npm publish + git tag still go through; only the GitHub Release page
stays empty.

### Bundle-size regression caught (v0.7.0)

```
You:  Verify
       (agent runs /verify → lint+typecheck+test+build pass →
        Step 5 Tier 1 publint clean → Tier 2 manual checks clean →
        Tier 3 npm pack --dry-run → Tier 4 compares unpackedSize to last
        cached version: prior was 52 KB, current is 128 KB → +146.7% delta)

⚠ Bundle-size regression on @ncbijs/eutils:
    Prior (v1.5.0):   52,100 bytes
    Current:         128,540 bytes
    Delta:           +146.7%

  Top 5 largest files in current tarball:
    dist/some-vendored-lib.js  72,431 bytes  (NEW vs prior — likely accidentally bundled)
    dist/index.js              31,200 bytes
    ...

  Investigate: did a dep get accidentally bundled? Did the build emit unminified output?

You:  [investigate, fix the build config, re-run /verify]
```

No auto-fix — bundle size deltas need human investigation.
First-run packages have no baseline; the cache builds organically over releases.
Major version bumps suppress the warning (1.5.0 → 2.0.0 expected to grow).

### One-stop morning health check (v0.7.0)

```
You:  How are my packages doing?
       (agent runs /solo-npm:status → renders the dashboard table →
        new "Portfolio health" section pulls audit + pkg-check counters from
        state.json cache — zero npm calls, instant render)

Portfolio health:
  Audit:      Tier-1: 0  Tier-2: 0      (last scan: 2d ago)
  Pkg-check:  errors: 0  warnings: 3    (last check: 6h ago)
  Action:     none — all clean.

You:  Tell me what's pending.
       (status's drift column flagged 2 unreleased commits; action hint
        suggests /release ready)
```

If a cache section is stale (>1d), the section shows "(stale — run /audit to refresh)"
instead of the values. To force-refresh, invoke `/solo-npm:status --fresh` —
runs /audit and /verify --pkg-check-only, then renders.

## Discoverability principles

These guide skill description tuning:

1. **Status uses portfolio + dashboard + drift keywords** — natural verbs users type for "how are my packages doing?".
2. **`/release` is the dominant entry** — auto-chain to prerelease means users mostly just type "release" or "ship" regardless of state.
3. **`/solo-npm:hotfix` recognises "patch", "previous major", "v1 line", "backport"** — natural phrasings for backward maintenance.
4. **One ambiguity to watch** — *"Update my deps"* is `/solo-npm:deps` (correct), but a less-tuned agent might think audit (also reads dep tree). Descriptions are tight: deps is "upgrade orchestrator", audit is "security audit".
5. **Compound workflows feel natural** — Day-1 bootstrap, routine release, monthly maintenance, major transition, hotfix all complete in 1–4 prompts of casual language. The user never opens a config file. solo-npm IS the workflow.
