---
description: Dependency upgrade orchestrator with /verify gates — classifies into tiers (trivial/safe/major/CVE), batches in dep-graph order, rolls back on failure. Triggers from prompts like "update my deps", "bump packages", "refresh dependencies", "catch up on dep versions", "apply CVE fixes", "upgrade typescript to v6", "don't break anything but update what's safe". Major upgrades require AskUserQuestion. Use monthly or on CVE.
---

# Deps

The chore solo devs skip. Dep upgrades are tedious, hostile to focus,
and risky. This skill orchestrates them with strict guardrails:
discover → classify → plan → apply (verify each batch) → commit →
rollback on failure.

## Phase 0 — Read prompt context

Scan the user's invoking prompt for hints that pre-fill subsequent decisions:

| Prompt mentions | Pre-fill |
|---|---|
| Specific dep names (`typescript`, `eslint`, `tar-fs`) | `TARGET_DEPS=[name1, name2, ...]` — limit `pnpm outdated` to those |
| `don't break anything`, `safe updates only`, `trivial only` | Pre-pick `Trivial only` tier; skip Phase 3's question |
| `update what's safe`, `safe stuff`, `safe updates` | Pre-pick `Trivial + Safe`; skip question |
| `all except major`, `no major bumps` | Pre-pick `All except Major`; skip question |
| `everything`, `all of it`, `all upgrades` | Pre-pick `All including Major` (still per-major gates fire) |
| `fix Tier 1`, `CVE fixes`, `apply CVEs from audit` | Pre-pick `cve-tier-1` filter (chained from /audit) |
| Specific target version (`upgrade typescript to v6`) | Set `TARGET_DEPS=[typescript]` AND target version 6; skip Phase 3's question |

Examples:
- *"Upgrade typescript to v6"* → `TARGET_DEPS=[typescript]`, target version 6. Skip tier prompt; Phase 3 renders 1-row plan; user clicks Proceed.
- *"Update what's safe"* → pre-pick Trivial+Safe; skip tier prompt.
- *"Fix Tier 1 from the audit"* → pre-pick `cve-tier-1`; skip tier prompt.
- Bare *"update deps"* → full classify + tier prompt.

If extraction is ambiguous, pre-fill what's clear and ask the rest. Don't over-extract.

## When to use

- Monthly maintenance: refresh deps proactively.
- Reactive: CVE landed, want to mitigate quickly.
- Pre-release sanity: "are my deps fresh before I ship?"
- After a long pause: "haven't touched deps in 6 months, what's outdated?"
- Chained from `/solo-npm:audit` Tier 1 fix.

## How it works

Six phases. Mutates `package.json` + lockfile, but every batch goes
through `/verify`; rollback on failure.

- **Phase 1** — discover (outdated + audit).
- **Phase 2** — classify into tiers.
- **Phase 3** — render plan + AskUserQuestion.
- **Phase 4** — apply (per batch with /verify gate, rollback on failure).
- **Phase 5** — commit per batch.
- **Phase 6** — done summary.

## Phase 1 — Discover

### 1a. Detect package manager + workspace shape

Same logic as `/solo-npm:audit` — pnpm-lock.yaml → pnpm; yarn.lock →
yarn; package-lock.json → npm.

Workspace shape:
- Single-package: just root `package.json`.
- pnpm workspace: `pnpm-workspace.yaml#packages` globs.
- npm/yarn workspace: `package.json#workspaces`.
- Nx monorepo: `pnpm-workspace.yaml` + `nx.json` — use `pnpm update --recursive`.

### 1b. Run outdated check

```bash
# pnpm
pnpm outdated --format json

# yarn 1.x
yarn outdated --json

# npm
npm outdated --json
```

For monorepos: `pnpm outdated --recursive --format json`. Aggregate
across workspaces; deduplicate (a dep used in many workspaces is one
upgrade decision, not many).

Each outdated entry has:
- `package`
- `current` (installed version)
- `wanted` (semver-resolved target)
- `latest` (registry latest)
- `dependencyType` (`production` | `development` | `peerDependency`)

### 1c. Run audit

```bash
pnpm audit --json
```

(Reuse output from `/solo-npm:audit` if invoked in the same session
with matching args — pass via context.)

Capture each advisory's `module_name`, `severity`, `fix_available`.

### 1d. Filter by argument (optional)

If invoked with a filter argument (e.g., `cve-tier-1` from /solo-npm:audit):

- `cve-tier-1`: only deps with critical+runtime+direct CVEs
- `cve-all`: all deps with any CVE
- `(no filter)`: all outdated deps

Default to "all outdated" when called without an argument.

## Phase 2 — Classify into tiers

For each candidate dep, classify:

### Tier: Trivial

- Patch bumps (`current.major == latest.major && current.minor == latest.minor`)
- Dev-only minor bumps (`dependencyType: development` AND `current.major == latest.major`)
- No CVEs

### Tier: Safe

- Runtime minor bumps (`current.major == latest.major`)
- No CVEs
- No `BREAKING CHANGE` indicator in lockfile peer-deps

### Tier: Major

- Any semver-major bump (`current.major < latest.major`)
- These are NEVER auto-applied; require human review of release notes

### Tier: CVE-driven

- Any dep with at least one open advisory (regardless of bump tier)
- Treated as urgent — applied before non-CVE deps even if Major

### Edge cases

- **Dep in Trivial AND CVE-driven**: classify as CVE-driven (apply with priority).
- **Peer dep mismatch after upgrade**: lockfile-resolution sometimes
  pulls a peer dep upgrade alongside. Treat the bundle as a single batch.
- **Workspace-internal deps** (e.g., `@ncbijs/etl` depending on
  `@ncbijs/utils`): skip; they're versioned together via the release flow.

## Phase 3 — Render plan + AskUserQuestion

Render the planned upgrades:

```
Outdated deps (excluding workspace-internal):

Trivial (N deps):
  - eslint              9.20.0 → 9.21.0     (patch)
  - prettier            3.5.1  → 3.5.2      (patch, dev)
  - vitest              2.0.5  → 2.1.0      (minor, dev)

Safe (M deps):
  - lodash              4.17.21 → 4.17.22   (patch, runtime)
  - semver              7.5.4   → 7.6.0     (minor, runtime)

Major (K deps — review needed):
  - typescript          5.4.5   → 6.0.0     (release notes: <url>)
  - eslint              9.21.0  → 10.0.1    (release notes: <url>)

CVE-driven (J deps):
  - tar-fs              2.1.0   → 2.1.4     (CVE-2024-XXXX, runtime)
  - qs                  6.5.2   → 6.10.3    (CVE-2024-YYYY, runtime)
```

For Major bumps, surface upstream release notes URL. Resolve via:

```bash
npm view <pkg> repository.url   # → github.com/owner/repo
gh release list --repo <owner>/<repo> --limit 5 --json tagName,name,url
```

Pick the release tag matching the target version.

Then call `AskUserQuestion`:

- header: `"Upgrade plan"`
- question: `"Apply which group?"`
- multiSelect: `false`
- options:
  1. `Trivial only` — patch bumps + dev-only minor (lowest risk)
  2. `Trivial + Safe` (Recommended) — runtime minor bumps too
  3. `All except Major` — adds CVE-driven, skips semver-major
  4. `All including Major` — full upgrade (be ready to read release notes)
  5. `Abort` — no changes

If `All including Major` is selected, do a follow-up
`AskUserQuestion` per Major dep:

- header: `"Major: <pkg>"`
- question: `"Apply <current> → <latest>?"`
- options: `Apply` / `Skip` / `Abort batch`

This is the human-decision gate for breaking changes. AI-orchestration
≠ AI-decision for breaking upgrades.

## Phase 4 — Apply (per batch with /verify gate)

Apply tiers in this order (lowest risk first):

1. CVE-driven (urgent, regardless of bump tier)
2. Trivial
3. Safe
4. Major (one at a time, gated)

Within each tier, batch in **dependency-graph order**: deps with no
internal dependencies first, then deps that depend on those, and so on.
This minimizes the chance of intermediate-state breakage.

For each batch:

### 4a. Snapshot

```bash
git stash --include-untracked --quiet --message "deps-batch-snapshot"
git stash pop --quiet  # immediately re-apply; this is just a checkpoint
# Alternative: git rev-parse HEAD > /tmp/deps-pre-batch-sha
```

Capture pre-batch state so rollback is precise.

### 4b. Update package.json + lockfile

```bash
# pnpm — single workspace
pnpm update <pkg-1>@<target> <pkg-2>@<target> ...

# pnpm — monorepo (recursive)
pnpm update --recursive <pkg-1>@<target> <pkg-2>@<target> ...

# yarn 1.x
yarn upgrade <pkg-1>@<target> <pkg-2>@<target>

# npm
npm install <pkg-1>@<target> <pkg-2>@<target>
```

For Major bumps, replace `@<target>` with `@<latest>` (already validated
by user's per-dep gate).

### 4c. Run /verify

```bash
# Invoke the project's verify wrapper if present, else /solo-npm:verify
/verify
# Or the agent invokes the Skill tool with name=verify or solo-npm:verify
```

Wait for the result. The verify skill returns `VERIFY_OK` or fails.

### 4d. Branch on /verify result (composes with agent-skills)

**On success**: continue to 4e (commit).

**On failure**: diagnose first, then decide.

```bash
git checkout package.json pnpm-lock.yaml
# Or for monorepo: git checkout package.json pnpm-lock.yaml packages/*/package.json
```

(The rollback happens before diagnosis to leave the working tree clean for analysis.)

**If `agent-skills@*` is installed** (check Claude Code's enabled plugins):

Invoke `/agent-skills:debugging-and-error-recovery` with the verify failure output as context. That skill brings reproduce → localise → reduce → fix → guard methodology, suited to "a dep upgrade broke the test suite". After diagnosis returns:

- If diagnosis identifies a single-dep cause: drop that dep from the batch, re-apply the rest (without it), re-verify. (Same outcome as the binary-search fallback below, but driven by analysis.)
- If diagnosis suggests the batch as a whole is incompatible: rollback stays; AskUserQuestion (Skip batch / Fix manually / Abort).

**If `agent-skills` is not installed** (fallback):

Surface the failure to the user and call `AskUserQuestion`:

- header: `"Verify failed"`
- question: `"How to handle <failed-batch-name>?"`
- options:
  1. `Skip this dep, continue` — drop the failing dep from the batch (binary-search if > 5 deps; one-at-a-time if ≤ 5), re-apply the rest, re-verify
  2. `Skip the whole batch, continue` — drop the entire batch, move to next tier
  3. `Fix manually then continue` — pause; user fixes, then types "continue" in chat
  4. `Abort all upgrades` — rollback all in-flight batches, leave committed batches in place

solo-npm works standalone via the fallback. Recommended setup is solo-npm + agent-skills installed together (see README "Scope and partners").

### 4e. Commit (only on success)

Stage `package.json` + lockfile + (monorepo) per-workspace
package.json files:

```bash
# Single-package
git add package.json pnpm-lock.yaml

# Monorepo
git add package.json pnpm-lock.yaml packages/*/package.json
```

Commit with a structured Conventional Commit message:

**Title** (one line, ≤ 72 chars):

```
chore(deps): bump <N> packages
```

For batches with 1-3 deps, list inline:

```
chore(deps): bump tar-fs 2.1.0 → 2.1.4, qs 6.5.2 → 6.10.3
```

**Body** (per dep, one line each):

```
- bump tar-fs from 2.1.0 to 2.1.4 (CVE-2024-XXXX)
- bump qs from 6.5.2 to 6.10.3 (CVE-2024-YYYY)
- bump lodash from 4.17.21 to 4.17.22
```

For CVE-driven entries, include the advisory ID in parentheses.

For Major bumps, include `BREAKING CHANGE: <reason>` in the body if
the user noted any deviation, otherwise just the bump line.

## Phase 5 — Push (gated)

After all batches are applied (or aborted), call `AskUserQuestion`:

- header: `"Push commits"`
- question: `"<N> commits ready. Push to origin?"`
- options:
  1. `Push now` (Recommended) — git push
  2. `Keep local` — user pushes manually later
  3. `Open in $EDITOR first` — `git rebase -i HEAD~<N>` for
     review (then user types "continue")

The "Push now" recommendation matches the AI-driven principle (agent does the operation; user gates). Power users with custom commit-amend workflows can pick option 2 or 3.

## Phase 6 — Done summary

Print:

```
Deps update complete.
  Applied:           <X> commits across <N> deps
  Skipped (verify):  <Y> deps
  Skipped (user):    <Z> deps
  Pending push:      <commits ready to push | none>

Suggested next steps:
  - /release ready?  (if commits are version-worthy)
  - /solo-npm:audit  (confirm CVEs are resolved)
```

## Phase 7 — Update audit cache

If any of the upgraded packages addressed advisories from the most recent `/solo-npm:audit` run (i.e., `cve-tier-1` filter was used, OR the upgrades resolved tier-1 entries based on package match):

1. Read `.solo-npm/state.json#audit`.
2. Recompute `tier1Count` (subtract resolved advisories) and `tier2Count` (similarly).
3. Save.

This keeps `/release` Phase A.5's audit cache consistent: a successful CVE-fix batch via /deps means the next /release won't STOP on the cached tier-1 count.

If no audit-related work was done, leave `cache.audit` untouched.

## Sharp edges

- **Lockfile thrash**: pnpm sometimes rewrites the lockfile on every
  invocation even when nothing actually changed. After
  `pnpm update <pkg>`, diff the lockfile via `git diff --stat
  pnpm-lock.yaml`. If the only changes are line reorderings (no version
  bumps), skip the commit and continue.

- **Monorepo workspace deps**: when a workspace's `package.json`
  references another workspace via `workspace:*`, that's an internal
  link. NEVER upgrade workspace-internal deps via this skill. Detect
  and exclude.

- **Concurrent CVE landing during batch**: if a new CVE alert lands
  mid-flow (rare but possible during long batch sequences),
  `pnpm audit` between batches will pick it up. Re-classify and re-plan
  if user accepts; otherwise continue with the original plan.

- **Peer dep failures**: pnpm by default warns on missing peer deps.
  An upgrade might bring in a new peer warning. /verify catches this
  if the project's lint config flags peer issues; otherwise treat as
  informational.

- **Major upgrade gate is non-negotiable.** AI-orchestration ≠
  AI-decision for breaking changes. Always surface release notes URL
  and call `AskUserQuestion` per Major dep.

- **Test thoroughly on Nx/pnpm workspaces.** Most demanding shape;
  workspace-internal deps + recursive update + per-package type
  classification. Validate that `pnpm update --recursive <pkg>` doesn't
  accidentally bump workspace-internal deps.

## What this skill does NOT do

- Does NOT skip the /verify gate. Every batch verifies; failures
  rollback.
- Does NOT auto-apply Major upgrades. Always per-dep AskUserQuestion.
- Does NOT push automatically. User confirms in Phase 5.
- Does NOT touch `pnpm-workspace.yaml`, `nx.json`, or other config
  files. Only `package.json` + lockfile.
- Does NOT remove deps. (`pnpm remove <pkg>` is out of scope.)
- Does NOT replace Dependabot. Dependabot opens PRs externally; this
  skill operates locally with verify gates. Different workflows; pick
  one.
- Does NOT track upgrade history. Future: `/solo-npm:deps --since 30d`.
