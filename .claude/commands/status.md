---
name: status
description: >
  One-page portfolio dashboard. Snapshot of all your published packages —
  latest version, last publish date, weekly downloads, OIDC trust state,
  recent commits, open issues, CI health. Read-only; no state changes.
  Use daily/weekly for a quick sanity check or portfolio review. Surfaces
  action hints (e.g., "@pkg has unreleased commits → /release ready?")
  but does not act on them.
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
provenance ✓).

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
```

The "unreleased commits" count is the **drift** signal: > 0 means main
is ahead of the latest published version.

### Trust state

Reuse the doctor's output from Phase 1. For each package, look at
`packages[].trustConfigured` (boolean) and `packages[].provenancePresent`
(boolean). The "Trust" column shows:

- `✓` if either trustConfigured or provenancePresent is true
- `—` if package is unpublished
- `✗` if published but neither trustConfigured nor provenancePresent

## Phase 3 — Render

### Main table

Render as a markdown table:

```
| Package          | Version  | Published    | Downloads/wk | Trust | CI  | Drift     |
|------------------|----------|--------------|-------------:|:-----:|:---:|-----------|
| @ncbijs/eutils   | 0.4.0    | 19d ago      |        1,213 | ✓     | ✓   | 2 commits |
| @ncbijs/blast    | —        | (unpublished)|            — | —     | —   | ready     |
| rfc-bcp47        | 0.13.0   | 22d ago      |          348 | ✓     | ✓   | clean     |
| npm-trust        | 0.9.0    | today        |           87 | ✓     | ✓   | clean     |
```

Columns:
- **Package**: package name (or `name@scope`)
- **Version**: latest on registry, or `—` for unpublished
- **Published**: relative time (`today`, `Nd ago`, `Nm ago`)
- **Downloads/wk**: from the unauthenticated downloads API
- **Trust**: `✓` configured; `—` unpublished; `✗` published-untrusted
- **CI**: latest workflow run conclusion (`✓` success, `✗` failure, `~`
  in_progress, `—` no runs)
- **Drift**: `clean`, `N commits` (unreleased), or `ready` (unpublished
  with local content)

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
| Drift > 0 commits | "→ /release ready?" |
| Unpublished package with local content | "→ /solo-npm:init for first publish?" |
| Unpublished package with no local tag | (no hint — not actionable) |
| Trust ✗ (published-untrusted) | "→ /solo-npm:trust to configure OIDC" |
| CI failure on latest run | "→ check `<gh run url>`; retry or fix" |
| Latest publish > 90 days ago | (no hint — informational only) |

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
