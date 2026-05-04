---
name: deps
description: >
  Curated dependency upgrade orchestrator with /verify gates. Detects
  outdated and vulnerable deps via `pnpm outdated` + `pnpm audit`,
  classifies into tiers (trivial/safe/major/CVE-driven), batches the
  safe upgrades in dependency-graph order, runs /verify after each
  batch, and rolls back on failure with an `AskUserQuestion` gate. One
  commit per batch with structured message. Major upgrades NEVER
  auto-applied — surfaces upstream release notes for human review. Use
  monthly for maintenance or reactively when CVEs land.
---

# Deps

The chore solo devs skip. Dep upgrades are tedious, hostile to focus,
and risky. This skill orchestrates them with strict guardrails:
discover → classify → plan → apply (verify each batch) → commit →
rollback on failure.

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

### 4d. Branch on /verify result

**On success**: continue to 4e (commit).

**On failure**: rollback this batch.

```bash
git checkout package.json pnpm-lock.yaml
# Or for monorepo: git checkout package.json pnpm-lock.yaml packages/*/package.json
```

Then surface the failure to the user and call `AskUserQuestion`:

- header: `"Verify failed"`
- question: `"How to handle <failed-batch-name>?"`
- options:
  1. `Skip this dep, continue` — drop the failing dep from the batch,
     re-apply the rest, re-verify
  2. `Skip the whole batch, continue` — drop the entire batch, move
     to next tier
  3. `Fix manually then continue` — pause; the user fixes, then
     types "continue" in chat to proceed (free-form)
  4. `Abort all upgrades` — rollback all in-flight batches, leave
     committed batches in place

If "Skip this dep, continue", reduce the batch to its working subset
(by binary search if the batch had > 5 deps, or one-at-a-time if ≤ 5).
This handles the common case of one bad apple in a batch.

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
  1. `Push now` — git push
  2. `Just keep them local` — user pushes manually later
  3. `Open them in $EDITOR first` — `git rebase -i HEAD~<N>` for
     review (then user types "continue")

Default: don't push. Let the user push when ready.

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
