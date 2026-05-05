---
description: One-page portfolio dashboard across all your published packages — version, last publish, weekly downloads, OIDC trust state, recent commits, drift (unreleased commits), open GitHub issues, CI health. Triggers from prompts like "how are my packages doing", "show me the dashboard", "what's pending release", "anything published recently", "morning check", "weekly check". Read-only.
---

# Status

Read-only snapshot of your npm portfolio. Replaces the "open 5 browser
tabs to check on my packages" morning ritual with one command.

## When to use

- Daily session start: "what's my portfolio doing?"
- Weekly review: "what shipped, what's pending?"
- Pre-release sanity: "are my other packages healthy before I ship this one?"
- After a CVE landing: "show me everything published recently — anything I
  need to bump?"

## How it works

Three phases. Read-only — no state changes, no commits, no API mutations.

- **Phase 1** — discover packages from the workspace (or an explicit `--scope`).
- **Phase 2** — fan out concurrent fetches: npm registry, downloads API,
  GitHub repo + CI, local git, npm-trust doctor.
- **Phase 3** — render a single markdown table + action hints.

## Phase 1 — Discover packages

Default: workspace-aware via `npm-trust --doctor --json` (already invoked
in Phase 2 for trust state — reuse the output).

```bash
pnpm exec npm-trust --doctor --json --workflow release.yml 2>/dev/null
```

The JSON's `workspace.packages[]` is the list. Each entry has `{ name,
version, dir, private }`. Skip entries with `private: true`.

If the workspace is single-package, the list has one entry.

If `--doctor` isn't available (older CLI), fall back to:

- Single-package: read `package.json#name`
- pnpm workspace: walk `pnpm-workspace.yaml#packages` globs, read each `package.json#name`
- npm/yarn workspace: walk `package.json#workspaces` globs

**Optional explicit scope mode**: if the user invokes `/solo-npm:status
--scope @myorg`, fetch the package list from the npm registry instead:

```bash
npm access list packages @myorg --json
```

(Useful for cross-cutting org-wide audits.)

## Phase 2 — Fan-out fetches

For each package, fetch in **parallel** (don't serialize):

### Per-package npm fetches

```bash
# Returns "version time.modified" — combined for one call:
npm view <pkg> --json
```

Parse: `version`, `time.modified`, `dist.attestations` (if present →
provenance ✓), and `dist-tags` (full map: `latest`, `next`, `v1`, `v2`, etc.).

The `dist-tags` map drives the dual-channel rendering in Phase 3:
- If only `latest` is set → single row.
- If `next` exists and points at a version different from `latest` → potentially active divergence (render second row if the `next` version is pre-release-shaped) or stale `next` (warning above table if `next` points at a superseded version).
- Legacy major dist-tags (`v1`, `v2`, …) → flagged in the Maintenance lines section, not the main table.

### Per-package downloads (unauthenticated, lenient)

```bash
curl -s "https://api.npmjs.org/downloads/point/last-week/<pkg>"
```

Returns `{ "downloads": <count>, "package": "<pkg>" }` or 404 (private
packages return 404 — show "—").

### Per-repo GitHub fetches

For each unique repo (dedup if multiple packages share a repo):

```bash
gh repo view <owner>/<repo> --json defaultBranchRef,issues,stargazerCount,url
gh run list --repo <owner>/<repo> --workflow release.yml --limit 1 \
   --json conclusion,headSha,startedAt,url
```

If the repo has > 30 published packages, batch with `gh api graphql`
instead.

### Per-repo local git

```bash
cd <repo_dir>
git log --since=7.days.ago --oneline
git tag --list 'v*' --sort=-v:refname | head -1   # latest tag
git rev-list <latest_tag>..HEAD --count           # unreleased commits

# Most recent stable tag (excludes pre-releases) — drives Maintenance-lines classification
git tag --list 'v*' --sort=-v:refname | grep -vE -- '-[a-z]+\.[0-9]+$' | head -1

# Maintenance branches on the remote (pattern: <major>.x)
git ls-remote --heads origin '*.x' | awk '{print $2}' | sed 's|refs/heads/||'

# Per maintenance branch: latest tag on that branch + age
for BRANCH in $(maintenance_branches); do
  MAJOR=$(echo "$BRANCH" | sed 's/\.x$//')
  TAG=$(git tag --list "v${MAJOR}.*" --sort=-v:refname --merged "origin/${BRANCH}" | head -1)
  AGE=$(git log -1 --format=%cr "$TAG")
done
```

The "unreleased commits" count is the **drift** signal: > 0 means main
is ahead of the latest published version.

Maintenance branches feed the new "Maintenance lines" section in Phase 3 (rendered only when ≥1 maintenance branch exists).

### Trust state — read from cache, no live npm calls

Read `.solo-npm/state.json` if it exists.

For each package in the dashboard:

- `✓` if the package name is in `cache.trust.configured` AND the cache is fresh (`now - cache.trust.lastFullCheck < cache.trust.ttlDays`)
- `—` if package is unpublished
- `✗` if the package is NOT in `cache.trust.configured` (cache is the source of truth; if a package isn't tracked it's assumed not-yet-trust-configured)

If the cache is stale (last full check > `ttlDays` ago) OR missing,
surface a one-line note above the table:

> ⚠️ Trust state cache stale (last full check N days ago). Run
> `/release` or `/solo-npm:trust` to refresh.

This skill is **read-only and fast** — it does NOT call `npm-trust
--doctor` to get live trust state. The cache is populated by `/release`
and `/solo-npm:trust`. Status's job is to *show* state, not derive it.

### Audit + pkg-check state — read from cache (NEW in v0.7.0)

Read `.solo-npm/state.json#audit` and `#pkgCheck` if present. These feed the new "Portfolio health" section in Phase 3.

```jsonc
// Expected shape:
{
  "audit": {
    "tier1Count": 0,
    "tier2Count": 0,
    "lastFullScan": "2026-05-04T11:00:00Z",
    "ttlDays": 1
  },
  "pkgCheck": {
    "errorCount": 0,
    "warningCount": 0,
    "lastCheck": "2026-05-04T11:00:00Z",
    "ttlDays": 1,
    "lastSize": { /* per-package-version size cache; not surfaced in status */ }
  }
}
```

For each section: compute freshness (`now - lastScan/lastCheck < ttlDays`) and pass to the Phase 3 renderer.

**Optional `--fresh` flag**: if the user invokes `/solo-npm:status --fresh`, skip cache reads and instead invoke `/solo-npm:audit` + `/solo-npm:verify --pkg-check-only` to get live values. This is opt-in because it costs npm calls + runs the pack audit; default behavior stays cached + fast.

This skill remains **read-only and fast** by default. Only `--fresh` triggers fresh runs.

## Phase 3 — Render

### Stale-@next warning (above table, conditional)

If any package has `dist-tags.next` pointing at a version that's been superseded by `dist-tags.latest` (i.e., `next` is a pre-release whose stable equivalent has already shipped to `latest`), surface a warning above the table:

```
⚠ @next is stale on @ncbijs/eutils (points at 1.6.0-beta.3, but @latest is 1.6.0).
⚠ @next is stale on @ncbijs/blast  (points at 1.6.0-beta.3, but @latest is 1.6.0).
   Cleanup: /solo-npm:dist-tag cleanup-stale
```

One line per affected package. The cleanup hint at the bottom invokes `/solo-npm:dist-tag` (skill #10) which handles the bulk `npm dist-tag rm` operation across all stale packages with one approval gate.

### Main table

Render as a markdown table. Each package contributes one row per *active* dist-tag (default: just `@latest`; second row if `@next` is currently active and divergent):

```
| Package        | Channel | Version       | Published    | Downloads/wk | Trust | CI  | Drift     |
|----------------|---------|---------------|--------------|-------------:|:-----:|:---:|-----------|
| @ncbijs/eutils | @latest | 1.5.0         | 30d ago      |        1,213 | ✓     | ✓   | clean     |
|                | @next   | 1.6.0-beta.3  | 2d ago       |          112 | ✓     | ✓   | 0 commits |
| @ncbijs/blast  | —       | —             | (unpublished)|            — | —     | —   | ready     |
| rfc-bcp47      | @latest | 0.13.0        | 22d ago      |          348 | ✓     | ✓   | clean     |
| npm-trust      | @latest | 0.9.0         | today        |           87 | ✓     | ✓   | clean     |
```

Columns:
- **Package**: package name (or `name@scope`); blank on continuation rows
- **Channel**: `@latest`, `@next`, or `—` for unpublished
- **Version**: version on the channel, or `—` for unpublished
- **Published**: relative time (`today`, `Nd ago`, `Nm ago`)
- **Downloads/wk**: from the unauthenticated downloads API (per-channel breakdown unavailable; render `—` for the `@next` continuation row OR show the package-level total on the first row only)
- **Trust**: `✓` configured; `—` unpublished; `✗` published-untrusted (per-package, not per-channel)
- **CI**: latest workflow run conclusion (`✓` success, `✗` failure, `~`
  in_progress, `—` no runs)
- **Drift**: per-channel — `clean`, `N commits` (unreleased commits since the channel's tag), or `ready` (unpublished with local content)

Render rules:
- **Single-channel package**: one row, `@latest` channel (or `—` if unpublished).
- **Active divergence** (`@next` exists, differs from `@latest`, AND is pre-release-shaped — i.e., `^\d+\.\d+\.\d+-[a-z]+\.\d+$`): two rows, `@latest` first, `@next` continuation row with blank Package cell.
- **Stale @next** (`@next` exists, differs from `@latest`, but is NOT pre-release-shaped — i.e., its stable equivalent has shipped): single `@latest` row only; surface in the stale-@next warning above the table.

### Open issues section

Below the table:

```
Open issues:
  - 1 in @ncbijs/eutils — "Citation parsing fails for…" (#14)
  - 0 in rfc-bcp47
  - 0 in npm-trust
```

Hide repos with 0 issues from the per-line list; instead summarize
`Open issues: 1 total across 1 repo` if all but one are clean.

### Recent activity section

```
Recent activity (last 7d):
  - rfc-bcp47:    3 commits, 0 releases
  - ncbijs:       9 commits, 1 release (v0.4.0 of @ncbijs/eutils)
  - npm-trust:    1 commit,  1 release (v0.9.0)
```

### Maintenance lines section (conditional)

Render only when ≥1 `<major>.x` branch exists on origin. Omit the section header entirely when there are none.

```
Maintenance lines:
  - 1.x (last patch v1.5.2, 14d ago) — current stable major
  - 0.x (last patch v0.9.5, 90d ago) — legacy, dist-tag @v0
```

Per-line classification:
- Compute `LATEST_STABLE_MAJOR` = major of the most recent stable tag (excludes pre-releases).
- For each `<major>.x` branch:
  - `<major> >= LATEST_STABLE_MAJOR` → **"current stable major"** (this line is still on `@latest`, even if main is in pre-release for the next major).
  - `<major> < LATEST_STABLE_MAJOR` → **"legacy, dist-tag @v<major>"** (a newer stable major exists; this line publishes patches to `@v<major>`).

Edge cases:
- Branch exists but no matching tag → render with `(no patches yet)` instead of "last patch v<X>".
- Multiple maintenance branches → list newest major first, descending.

### Portfolio health section (NEW in v0.7.0)

Render after Maintenance lines (or after Recent activity if no maintenance lines exist), before Action hints. Always renders — no conditional.

Reads from `.solo-npm/state.json#audit` and `#pkgCheck` (cached values from `/audit` and `/verify --pkg-check-only` runs). Default behavior: zero npm calls; just reads JSON.

**Layout**:

```
Portfolio health:
  Audit:      Tier-1: 0  Tier-2: 0      (last scan: 2d ago)
  Pkg-check:  errors: 0  warnings: 3    (last check: 6h ago)
  Action:     none — all clean.
```

**Stale-cache rendering**: if a section's cache is stale (`now - lastScan > ttlDays`), substitute the values with a refresh hint:

```
Portfolio health:
  Audit:      (stale — last scan 9d ago. Run /solo-npm:audit to refresh.)
  Pkg-check:  errors: 0  warnings: 3    (last check: 6h ago)
  Action:     refresh audit cache to see current security state.
```

**Missing-cache rendering**: if a section has never been populated (e.g., fresh repo, never ran /audit):

```
Portfolio health:
  Audit:      (no scan yet — run /solo-npm:audit)
  Pkg-check:  (no scan yet — run /solo-npm:verify --pkg-check-only)
  Action:     run /solo-npm:audit and /solo-npm:verify --pkg-check-only to populate health cache.
```

**Action line variants**:

| State | Action line |
|---|---|
| All clean (zero counts, fresh caches) | `none — all clean.` |
| Tier-1 > 0 | `→ /solo-npm:audit (Tier-1 advisories require attention)` |
| Tier-2 > 0 (no Tier-1) | `→ /solo-npm:audit (Tier-2 advisories — review when convenient)` |
| pkg-check errors > 0 | `→ /solo-npm:verify --pkg-check-only (manifest errors block release)` |
| pkg-check warnings only | `→ /solo-npm:verify --pkg-check-only (manifest warnings — review)` |
| Multiple sections need attention | List each on its own line |

**`--fresh` flag (opt-in)**: when invoked as `/solo-npm:status --fresh`:
1. Phase 2 invokes `/solo-npm:audit` (which writes back to `.solo-npm/state.json#audit`).
2. Phase 2 invokes `/solo-npm:verify --pkg-check-only` (writes back to `#pkgCheck`).
3. Phase 3 renders fresh values.

The `--fresh` mode trades zero-cost speed for accuracy. Default mode is cache-only.

### Action hints section

For each detected drift, surface an actionable hint:

```
Action hints:
  • @ncbijs/eutils has 2 unreleased commits in ncbijs → /release ready?
  • @ncbijs/blast is local-only at v0.1.0 → /solo-npm:init for first publish?
```

Hint mapping:

| Detected | Hint |
|---|---|
| Drift > 0 commits on `@latest` row | "→ /release ready?" |
| Drift > 0 commits on `@next` row | "→ /solo-npm:prerelease (Bump or Promote)" |
| Active `@next` row, no drift | "→ /solo-npm:prerelease to bump or promote" |
| Unpublished package with local content | "→ /solo-npm:init for first publish?" |
| Unpublished package with no local tag | (no hint — not actionable) |
| Trust ✗ (published-untrusted) | "→ /solo-npm:trust to configure OIDC" |
| CI failure on latest run | "→ check `<gh run url>`; retry or fix" |
| Latest publish > 90 days ago | (no hint — informational only) |
| Legacy maintenance line, last patch > 365 days ago | "→ Consider archiving `<major>.x` — last patch was 11mo ago" (informational, no skill chain) |
| Stale `@next` (covered by stale-@next warning above table) | (no action hint — npm registry hygiene is outside solo-npm scope) |

## Output behavior

This skill is read-only. Do NOT call `AskUserQuestion`; do NOT chain
into other skills. Just render the dashboard + hints. The user decides
what to do next.

If the user explicitly types `/solo-npm:status` and then immediately
invokes another skill (e.g., `/release`), each skill is invoked
independently — `status` does not orchestrate the follow-up.

## Sharp edges

- **GitHub API rate limits.** Cap per-invocation at ~30 repos. If more,
  fall back to a `gh api graphql` bulk query for issue + CI counts. The
  skill should detect rate-limit errors and surface them rather than
  silently failing.
- **Private packages** return 404 from the public downloads API. Render
  `—` for downloads on private packages.
- **No `gh` auth.** If `gh auth status` fails, surface a one-line
  notice ("`gh` is not authenticated; CI + issue columns will show `?`")
  and continue with the rest of the data.
- **Slow first run.** With ~15 packages × ~3 fetches each, expect 5-10s
  on a fresh run. The npm and downloads APIs are unauthenticated and
  fast; `gh` calls dominate.
- **Stale cache.** v0.5.0 doesn't cache — every invocation fetches
  fresh. Future: add `--cache 1h` flag for re-runs in the same session.

## What this skill does NOT do

- Does NOT mutate state. No commits, no publishes, no `gh issue close`.
- Does NOT chain into other skills. Just suggests next-step skills via
  hints.
- Does NOT fetch private/auth-gated data beyond what `gh auth` already
  has access to.
- Does NOT replace `gh dash` or other rich-UI dashboards — it's
  terminal-friendly markdown.
- Does NOT track historical trends across runs (that would require a
  cache layer).
