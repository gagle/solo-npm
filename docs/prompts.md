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
        Bump/Promote/Abort → user picks Promote → publishes 2.0.0 to @latest)
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
       (forward-port is currently manual — user describes the fix again on main,
        then /release auto-chains to /solo-npm:prerelease → bump to 2.0.0-beta.5)
```

(Future: forward-porting becomes one prompt — out of scope for v0.5.4.)

### CVE landed in popular dep

```
You:  I saw the GHSA-xxxx CVE in tar-fs. Audit + fix.
       (agent runs /solo-npm:audit → confirms repo affected → tier 1 entry →
        chains into /solo-npm:deps cve-tier-1 → upgrades tar-fs → /verify → commit)
You:  Ship the patch to all affected packages.
       (agent runs /release → patch bump → v0.5.5 to @latest)
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

## Discoverability principles

These guide skill description tuning:

1. **Status uses portfolio + dashboard + drift keywords** — natural verbs users type for "how are my packages doing?".
2. **`/release` is the dominant entry** — auto-chain to prerelease means users mostly just type "release" or "ship" regardless of state.
3. **`/solo-npm:hotfix` recognises "patch", "previous major", "v1 line", "backport"** — natural phrasings for backward maintenance.
4. **One ambiguity to watch** — *"Update my deps"* is `/solo-npm:deps` (correct), but a less-tuned agent might think audit (also reads dep tree). Descriptions are tight: deps is "upgrade orchestrator", audit is "security audit".
5. **Compound workflows feel natural** — Day-1 bootstrap, routine release, monthly maintenance, major transition, hotfix all complete in 1–4 prompts of casual language. The user never opens a config file. solo-npm IS the workflow.
