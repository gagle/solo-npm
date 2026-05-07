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

## Error-handling patterns (H2, H8 from `/unpublish` reference)

**H8 rate-limit backoff (v0.12.0)**: `/status` Phase 2 fans out parallel `npm view` calls — one per package — and is the highest-impact application of H8. Every parallel `npm view` invocation must be wrapped via the `npm_with_h8_backoff` helper from `/unpublish` Phase −1.9. On 429 / "rate-limited" stderr, retry with exponential 1s/2s/4s/8s + ±20% jitter; max 4 attempts; surface "rate-limited; retry later" per-package if exhausted. The dashboard renders that package's row as `(rate-limited)` instead of `—` so the user knows it was throttled vs unavailable.

**SSL/TLS error remediation (v0.12.0)**: every external HTTPS call (curl to deps.dev, npm view to registry, gh repo view) checks stderr against the SSL pattern set in `/unpublish` Phase −1.10. Non-fatal at the per-call level — that package's row falls back to `—` with a one-line "SSL error; see remediation hints below table" note that includes the corporate-proxy / transient / OS-CA-bundle / `--insecure` last-resort guidance once at the bottom of the dashboard.

**H2 corruption guard** (carry-forward):

`/status` reads `.solo-npm/state.json` for trust, audit, pkgCheck, and depsdev cache sections (Phase 2 below). Wrap every `JSON.parse(state.json)` read in try/catch — on parse fail, surface non-fatal warning *".solo-npm/state.json is malformed; rendering with empty cache (some columns may show `—` or `(no scan yet)`). Remove the file and re-run any solo-npm skill to regenerate."* and continue rendering with empty defaults per cache section. Don't crash the dashboard on a single malformed entry.

`/status` is read-only and doesn't auto-chain into other skills (just suggests them as next-steps), so H6 (chain-target failure recovery) doesn't apply here. The `--fresh` flag manually invokes `/audit` and `/verify --pkg-check-only`; if those STOP, surface their diagnostics verbatim before falling back to render the dashboard with whatever cache state existed before.

## Phase 2 — Fan-out fetches

For each package, fetch in **parallel** (don't serialize):

### Per-package npm fetches

```bash
# Returns "version time.modified" — combined for one call:
# 2>/dev/null prevents npm warnings from corrupting the JSON parse.
# timeout 30 bounds against a hung/slow registry.
timeout 30 npm view <pkg> --json 2>/dev/null
```

Parse: `version`, `time.modified`, `dist.attestations` (if present →
provenance ✓), and `dist-tags` (full map: `latest`, `next`, `v1`, `v2`, etc.).

The `dist-tags` map drives the dual-channel rendering in Phase 3:
- If only `latest` is set → single row.
- If `next` exists and points at a version different from `latest` → potentially active divergence (render second row if the `next` version is pre-release-shaped) or stale `next` (warning above table if `next` points at a superseded version).
- Legacy major dist-tags (`v1`, `v2`, …) → flagged in the Maintenance lines section, not the main table.

### Per-package downloads (unauthenticated, lenient)

```bash
curl -s --max-time 10 --connect-timeout 5 "https://api.npmjs.org/downloads/point/last-week/<pkg>"
```

Returns `{ "downloads": <count>, "package": "<pkg>" }` or 404 (private
packages return 404 — show "—"). On timeout (curl exit 28), render `—` and continue;
the dashboard is read-only and partial data is fine. Don't HARD STOP the whole `/status`
on one slow API call.

### Per-repo GitHub fetches

For each unique repo (dedup if multiple packages share a repo):

```bash
gh repo view <owner>/<repo> --json defaultBranchRef,issues,stargazerCount,url
gh run list --repo <owner>/<repo> --workflow release.yml --limit 1 \
   --json conclusion,headSha,startedAt,url
```

### gh GraphQL fan-in for >20-package portfolios (Tier-3 I, v0.12.0)

For portfolios with more than 20 unique repos, the per-repo fan-out above produces 2N gh API calls — slow + risks rate limits. Switch to a single `gh api graphql` call that fetches all repos in one round-trip:

```bash
# Threshold: > 20 unique repos triggers GraphQL fan-in.
UNIQUE_REPOS=$(echo "$REPO_LIST" | sort -u | wc -l)
if [ "$UNIQUE_REPOS" -gt 20 ]; then
  # Build the GraphQL query with N repository(name:..., owner:...) sub-queries (aliased)
  QUERY='query {'
  i=0
  for repo in $REPO_LIST_DEDUP; do
    OWNER="${repo%/*}"
    NAME="${repo#*/}"
    QUERY+="
      r${i}: repository(owner: \"${OWNER}\", name: \"${NAME}\") {
        nameWithOwner
        defaultBranchRef { name }
        latestRelease { tagName publishedAt }
        openIssues: issues(states: OPEN, first: 1) { totalCount }
        openPRs: pullRequests(states: OPEN, first: 1) { totalCount }
        url
      }"
    i=$((i + 1))
  done
  QUERY+='
}'

  # Single API call — captures all repo metadata in one round-trip
  RESULT=$(timeout 30 gh api graphql -f query="$QUERY" 2>&1)
  GRAPHQL_RC=$?

  if [ $GRAPHQL_RC -ne 0 ]; then
    # Common failures: insufficient token scope, rate-limited at the global level, malformed query
    echo "WARN: gh api graphql fan-in failed: $RESULT"
    echo "      Falling back to per-repo individual gh repo view + gh run list calls."
    # The per-repo fallback below applies the H8 rate-limit backoff helper as well.
    USE_GRAPHQL=0
  else
    # Parse RESULT.data.r0, .r1, ... and merge into the per-repo data structure used by Phase 3.
    USE_GRAPHQL=1
  fi
fi

if [ "${USE_GRAPHQL:-0}" -ne 1 ]; then
  # Per-repo fallback (the original block above) — applied for ≤20 repos OR after graphql failure.
  # Each call goes through the H8 rate-limit backoff helper from /unpublish Phase −1.9.
  for repo in $REPO_LIST_DEDUP; do
    npm_with_h8_backoff "gh repo view $repo --json defaultBranchRef,issues,stargazerCount,url"
    npm_with_h8_backoff "gh run list --repo $repo --workflow release.yml --limit 1 --json conclusion,headSha,startedAt,url"
  done
fi
```

The 20-package threshold is a heuristic — below it, the latency savings of GraphQL don't outweigh the complexity (and rate-limit risk for unauthenticated `gh` is 60req/hr, easily blown by a 30-pkg portfolio). The user's typical portfolio is ≤10 repos so the fallback path is the common case.

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

**H6 chain-failure recovery on `--fresh` (v0.11.0)**: if either `/audit` or `/verify --pkg-check-only` STOPs internally (audit registry unreachable; verify halts on secrets-detected; etc.), capture the diagnostic and **fall back to rendering the dashboard from existing cache** with a clear note:

```
NOTE: --fresh requested but the live refresh stopped.
  /solo-npm:audit   — <verbatim child error>
  /solo-npm:verify  — <verbatim child error or "ok">

Rendering dashboard from existing cache (last refreshed N days ago).
Run /solo-npm:audit / /solo-npm:verify directly to investigate the failure.
```

The dashboard is informational; partial-fresh data plus stale-fallback is more useful than a hard failure that produces no output.

This skill remains **read-only and fast** by default. Only `--fresh` triggers fresh runs.

### deps.dev signals — fetch + cache (NEW in v0.9.0)

For each package, fetch security + posture metadata from Google's [deps.dev API](https://docs.deps.dev/api/v3/) (free, no authentication required). The API aggregates npm registry data with OSV.dev advisories and OpenSSF Scorecard ingestion.

```bash
# Per-package fetch — JSON over HTTP:
curl -s --max-time 10 --connect-timeout 5 "https://api.deps.dev/v3/systems/npm/packages/<pkg-name>"
```

On timeout/network error: curl exits non-zero, jq receives empty input. Render the security-signals row as
`(deps.dev unavailable)` and continue. As with downloads above, don't block the dashboard on a slow API.

Parse:
- `defaultVersion` → most recent published version
- `versions[].publishedAt` → publish timestamps
- `versions[].advisoryKeys[]` → OSV vulnerability IDs
- `versions[].licenses[]` → license SPDX identifiers
- `scorecard` (when present) → OpenSSF Scorecard score `[0-10]` and check breakdown

**Caching**: write to `.solo-npm/state.json#depsdev`:

```jsonc
{
  "depsdev": {
    "lastFullScan": "2026-05-06T11:00:00Z",
    "ttlDays": 1,
    "perPackage": {
      "@ncbijs/eutils": {
        "scorecardScore": 7.8,
        "advisoryCount": 0,
        "license": "MIT",
        "lastFetched": "2026-05-06T11:00:00Z"
      }
    }
  }
}
```

TTL: 1 day. The deps.dev data updates lazily; daily refresh is sufficient.

**Failure handling**: deps.dev is a free public API. If it's unreachable (network down, API rate-limit hit, or service deprecation), surface `depsdev: (unavailable)` in the Phase 3 render and continue with the rest of /status. **Do not block the dashboard on deps.dev availability.**

**Skip if `--fresh` is NOT passed and cache is fresh**: read from `.solo-npm/state.json#depsdev.perPackage` instead of refetching. Same hot-path optimization as audit + pkg-check.

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

### Security signals section (NEW in v0.9.0)

Render after Portfolio health, before Action hints. Renders only when deps.dev cache is populated for at least one package; otherwise omitted entirely.

Pulls from `.solo-npm/state.json#depsdev.perPackage` — per-package OSV advisory counts, OpenSSF Scorecard score (0–10), license SPDX identifier.

**Layout**:

```
Security signals (deps.dev):
  Package           Score   Advisories   License
  @ncbijs/eutils    7.8     0            MIT
  @ncbijs/blast     7.8     0            MIT
  rfc-bcp47         8.4     0            Apache-2.0
  npm-trust         6.2     1            MIT

  Source: api.deps.dev (last fetched: 2h ago).
```

**Score interpretation** (OpenSSF Scorecard):
- 8–10: strong security posture (signed releases, branch protection, vuln-reporting policy, dependency-update automation, etc.)
- 5–7: typical for solo-dev OSS (some checks fail because they assume team workflows like code review)
- 0–4: posture concerns; investigate

**Advisory count**: distinct OSV advisory IDs affecting the most-recent published version. Cross-references with /audit's tier counts (which are derived differently — npm/pnpm audit output, not OSV directly).

**Stale cache rendering**: if `now - lastFullScan > ttlDays`, append a stale hint:
```
  (cache stale — last fetched 4d ago. Run /solo-npm:status --fresh to refresh.)
```

**Unavailable rendering**: if deps.dev was unreachable on the last fetch attempt (cache-flagged as unreachable), substitute the table with:
```
Security signals (deps.dev):
  (deps.dev unavailable — couldn't fetch security signals. Try again later or /solo-npm:status --fresh.)
```

This section gives a quick read on package posture without invoking `/audit` (which is heavier — runs pnpm audit). Complementary signal: `/audit` runs your local audit workflow; deps.dev gives the OSV.dev community's view.

**Action hint integration**: if any package has Scorecard score < 5 OR advisory count > 0, the hint section below picks up the package and suggests `/solo-npm:audit` for a deeper look.

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
