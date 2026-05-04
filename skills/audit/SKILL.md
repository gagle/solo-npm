---
name: audit
description: >
  Security audit across the workspace. Runs `npm/pnpm audit`, classifies
  advisories by exploit risk (critical/high/moderate/low) and dep type
  (runtime vs dev, direct vs transitive), and surfaces only the actionable
  subset. Suggests fix paths: in-place upgrade (chains to /solo-npm:deps),
  transitive override, or wait for upstream patch. Read-only — does not
  apply fixes itself. Use monthly or when an advisory lands.
---

# Audit

Read-only security audit with risk triage. The default `npm audit` /
`pnpm audit` output is a firehose; this skill classifies it into
actionable tiers so a solo dev knows what to fix today vs. what to defer.

## When to use

- Monthly maintenance ritual.
- Reactive: a CVE just landed, want to know if you're affected.
- Pre-release sanity: "anything I should bump before shipping?"
- Post-`/solo-npm:deps`: confirm the upgrade resolved the alerts.

## How it works

Five phases. Read-only — no mutations.

- **Phase 1** — run the audit (auto-detect package manager).
- **Phase 2** — enrich each advisory with dep-tree context.
- **Phase 3** — classify into 4 tiers.
- **Phase 4** — render per-tier tables + brief trust audit.
- **Phase 5** — gate with `AskUserQuestion` if tier 1 or 2 has entries;
  optionally chain into `/solo-npm:deps`.

## Phase 1 — Run audit

Auto-detect package manager:

| File | Manager |
|---|---|
| `pnpm-lock.yaml` | pnpm |
| `yarn.lock` | yarn |
| `package-lock.json` | npm |
| (none) | npm (default) |

Run:

```bash
# pnpm
pnpm audit --json

# yarn (yarn 1.x)
yarn audit --json

# npm
npm audit --json
```

Capture full advisory list. Each advisory has at minimum:
- `severity` (critical | high | moderate | low | info)
- `module_name` (the vulnerable package)
- `vulnerable_versions` / `fix_available`
- `path` (dep tree path: `root>parent>vulnerable_pkg`)
- `url` (GitHub Security Advisory or CVE link)

Different package managers expose this differently. Normalize to a
common shape internally.

## Phase 2 — Enrich

For each advisory, compute:

### Dep-tree depth

Walk the lockfile (or audit output's `path`):
- **Direct**: vulnerable package is in `package.json#dependencies` or
  `devDependencies`.
- **Transitive**: vulnerable package is a sub-dep of a direct dep.

### Dep type (runtime vs dev)

If the path's root is in `package.json#dependencies`, it's **runtime**.
If the root is in `devDependencies`, it's **dev-only**.

(For monorepos: walk each workspace's `package.json` to determine type
per workspace; an advisory might be runtime in one workspace and dev in
another — flag both.)

### Fix availability

From audit JSON's `fix_available`:
- `true` (auto-fixable): a non-breaking upgrade resolves it.
- `{ name, version, isSemVerMajor }`: a specific upgrade is needed; if
  `isSemVerMajor: true`, requires manual review.
- `false`: no upgrade path yet (upstream patch needed, or transitive
  with no override path).

## Phase 3 — Classify into tiers

Apply this matrix:

| Severity | Type | Direct/Transitive | Fix path | Tier |
|---|---|---|---|---|
| critical | runtime | direct | available | **Tier 1 — Fix today** |
| critical | runtime | transitive | available (override or upgrade ancestor) | **Tier 1 — Fix today** |
| critical | runtime | transitive | not available | **Tier 2 — Plan upgrade / monitor upstream** |
| critical | dev | any | any | **Tier 3 — Lower priority** |
| high | runtime | direct | available | **Tier 2 — Plan upgrade** |
| high | runtime | transitive | available | **Tier 2 — Plan upgrade** |
| high | runtime | transitive | not available | **Tier 4 — Note** |
| high | dev | any | any | **Tier 3 — Lower priority** |
| moderate | runtime | direct/transitive | any | **Tier 3 — Lower priority** |
| moderate | dev | any | any | **Tier 4 — Note** |
| low | any | any | any | **Tier 4 — Note** |
| info | any | any | any | **Tier 4 — Note** |

Edge case: if the same package appears in multiple workspaces with
different types (runtime in one, dev in another), classify by the
**most severe** combination.

## Phase 4 — Render

### Trust audit prefix

At the top of the report, surface trust state from `npm-trust --doctor`:

```
Trust audit:
  ✓ All N packages have OIDC trust + provenance.
```

Or:

```
Trust audit:
  ✗ M of N packages need trust → /solo-npm:trust
    - @scope/pkg-a (no trust, no provenance)
    - @scope/pkg-b (no trust, no provenance)
```

This is a brief cross-cutting check, NOT a deep config. For deeper
trust state inspection, use `/solo-npm:trust`.

### Per-tier tables

Render each tier as a separate markdown table. Tier 1 and 2 first; 3
and 4 below in a collapsed section if there are many entries.

```
## Tier 1 — Fix today (critical, runtime, direct)

| Advisory       | Package        | Path                        | Fix in |
|----------------|----------------|-----------------------------|--------|
| GHSA-1abc-...  | tar-fs 2.1.0   | direct (production)         | 2.1.4  |
| GHSA-2def-...  | qs 6.5.2       | direct (production)         | 6.10.3 |

## Tier 2 — Plan upgrade (high, runtime; or transitive critical)

| Advisory       | Package        | Path                        | Fix in |
|----------------|----------------|-----------------------------|--------|
| GHSA-3ghi-...  | semver 5.7.1   | direct (production)         | 7.5.4  |
| GHSA-4jkl-...  | minimist 1.2.5 | parent: mocha (transitive)  | 1.2.8 (via mocha 10) |

## Tier 3 — Lower priority (dev-only or moderate runtime)

(N entries — expand to see)

## Tier 4 — Note, don't act (low/info or no upgrade path)

(N entries — expand to see)
```

The "Path" column shows the chain: `direct (production)`, `direct
(dev)`, or `parent: <pkg> (transitive)`.

The "Fix in" column shows the upgrade target. For transitive vulns
that need an ancestor upgrade, format as `<version> (via
<ancestor> <ancestor_version>)`.

### Summary line

```
Summary: 2 critical, 4 high, 8 moderate, 12 low across 23 packages.
Action: 2 in Tier 1, 6 in Tier 2.
```

## Phase 5 — Gate (only if Tier 1 or 2 has entries)

If the report shows Tier 1 or 2 advisories, call `AskUserQuestion`:

- header: `"Audit"`
- question: `"How do you want to handle <N> Tier 1 + <M> Tier 2 advisories?"`
- multiSelect: `false`
- options:
  1. `Fix Tier 1 now` (Recommended) — invoke `/solo-npm:deps` filtered
     to the CVE-tier-1 packages
  2. `Fix Tier 1 + 2` — invoke `/solo-npm:deps` filtered to all
     auto-fixable advisories
  3. `Show me details` — print full advisory text inline; no action
  4. `Defer` — acknowledge; no action

If Tier 1 + 2 are empty, skip the gate; the audit is complete after
Phase 4.

## Composition with /solo-npm:deps

When the user picks "Fix Tier 1 now" or "Fix Tier 1 + 2", invoke
`/solo-npm:deps` with a filter argument identifying which advisories
to address:

```
/solo-npm:deps cve-tier-1
```

The deps skill recognizes this filter and limits its discovery to
packages with CVE advisories in the requested tier. (Implementation
detail: deps re-runs `pnpm audit` itself; the filter is a hint.)

## Sharp edges

- **False-positive transitive vulns.** Many advisories are for
  transitive deps with no upgrade path or in dev-only contexts. The
  tier classifier puts these in Tier 4 ("note, don't act") so the
  actionable subset is small. If you find a real CVE incorrectly in
  Tier 4, it's likely because the lockfile shows no fix available —
  that means upstream needs to publish a patch.
- **GHSA mapping** vs. CVE numbers. npm/pnpm audit returns GHSA IDs;
  if the user asks "what about CVE-2024-XXXX", grep the audit JSON's
  `cves` field per advisory.
- **Yarn 1.x audit JSON shape** differs from pnpm/npm. If the user is
  on yarn 1.x, normalize by parsing line-delimited JSON instead of a
  single object.
- **Workspace audits**: `pnpm audit --recursive` covers all workspaces;
  `npm audit` historically only audits the root. Detect workspace mode
  and use the appropriate flag.
- **The skill is read-only — it never applies fixes.** All upgrades go
  through `/solo-npm:deps` (which has the verify gates). This is
  intentional: audit shows risk; deps mitigates with safety.

## What this skill does NOT do

- Does NOT apply fixes. (That's `/solo-npm:deps`.)
- Does NOT mutate `package.json` or lockfile.
- Does NOT call out to security advisory APIs beyond what npm/pnpm
  audit already does.
- Does NOT include compliance audits (license, SBOM). Out of scope for
  v0.5.0.
- Does NOT track historical advisories across runs (no cache).
