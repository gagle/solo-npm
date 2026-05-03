# release-solo-npm

> Tag-triggered npm releases with one approval, for AI-driven solo dev
> where PRs are disabled.

## The problem

You're a solo developer — or running a small group of LLM agents —
shipping an npm package. PRs are disabled in your repo
(issue/discussion contribution model only). There's no committee to
review, no second pair of human eyes.

Existing release tooling is built for teams: PR-based workflows,
multi-stage approvals, complex changelog negotiation. In a solo or
agent-driven context, that overhead becomes friction — and friction
makes you skip steps when you're moving fast. Skipped steps make
unsigned, unverified, opaque releases.

You want releases that are:

- **Boring**: tested, typed, provenanced, every time. No exceptions.
- **One-touch**: not a 15-step runbook, not a commit-message lottery.
- **Provable**: SLSA attestations on every tarball, traceable to git.

## The solution

A Claude Code skill that drives the entire release end-to-end:

| Phase | What happens | Human input |
|---|---|---|
| **A. Pre-flight** | `/verify-solo-npm` (lint + test + build) → `npm-trust --doctor` | none (silent if green) |
| **B. Plan** | Detect bump, render summary, **one** `AskUserQuestion` selector | one click: Proceed / Override / Edit changelog / Abort |
| **C. Execute** | Bump → commit → push → tag → watch CI → verify on registry | none |

Type `/release-solo-npm`. Review the plan once. Click `Proceed`. Get
a notification when the tarball is on npm with provenance. That's it.

## Why this works for solo / agent-driven dev

- **Trust replaces review.** Tag-triggered CI runs the same gates the
  agent ran locally; SLSA provenance proves what was published.
- **One structured approval gate.** Not a free-text "yes/no" prompt
  (those are easy to miss). A clearly-labeled selector — unmissable.
- **Composable.** `/verify-solo-npm` is its own skill. Customize it
  (add e2e, coverage thresholds, license audits) without touching
  `/release-solo-npm`.
- **No special cases.** Every release follows the same path. The
  "small fix" and "big feature" both go through Phase A → B → C.

## Install

```
/plugin marketplace add gagle/release-solo-npm
/plugin install release-solo-npm@gllamas-skills
```

This installs both `/release-solo-npm` and `/verify-solo-npm` skills
into Claude Code.

## Prerequisites

- A tag-triggered `release.yml` workflow that publishes via
  `pnpm publish --no-git-checks` and reads `publishConfig` from
  `package.json`.
- OIDC trust configured for the package (one-time setup via
  [`gagle/npm-trust`](https://github.com/gagle/npm-trust) and the
  bundled `setup-npm-trust` skill).
- Conventional commit messages (`feat:`, `fix:`, etc.) since the
  release skill parses them to compute the version bump.

## Configuration

After install, edit `.claude/skills/release-solo-npm/SKILL.md` to fill
in the placeholders for your repo:

| Placeholder | Example | What it controls |
|---|---|---|
| `<PACKAGE_NAME>` | `rfc-bcp47` | npm view target |
| `<REPO_SLUG>` | `gagle/rfc-bcp47` | compare URLs in CHANGELOG |
| `<RELEASE_WORKFLOW>` | `release.yml` | doctor's `--workflow` filter |
| Monorepo block | (delete if single-package) | iterate `packages/*` |
| `<LINT_CMD>`, `<TEST_CMD>`, `<BUILD_CMD>` | `pnpm run lint`, `pnpm test`, `pnpm run build` | Fallbacks if `/verify-solo-npm` isn't installed |

Customize `.claude/skills/verify-solo-npm/SKILL.md` for your stack —
add e2e, coverage gates, typecheck, whatever.

## Pre-release versions

The skill detects `x.y.z-pre.n` shape automatically. Phase B's
`AskUserQuestion` adapts to:

- `Bump pre-release counter` (e.g., `1.3.0-beta.1` → `1.3.0-beta.2`)
- `Promote to stable` (`1.3.0-beta.n` → `1.3.0`)
- `Override` / `Abort`

To start a pre-release line from stable, use `Override version` and
type something like `1.3.0-beta.1` — the skill accepts it verbatim.

## Custom / private registries

Set `publishConfig.registry` to your registry URL in `package.json`.
The skill detects this and skips the provenance verification step
(SLSA only works on public npm).

```json
"publishConfig": {
  "access": "public",
  "registry": "https://npm.your-internal-registry.example.com/"
}
```

Note: do NOT include `"provenance": true` for custom registries.
`npm-trust --doctor` will flag the misconfig (`REGISTRY_PROVENANCE_CONFLICT`).

## Monorepos

The bundled `release-solo-npm/SKILL.md` ships with a monorepo block.
Single-package repos delete it after install. Monorepos keep it;
Phase C iterates `packages/*` for the version bump and per-package
registry verification.

See [`gagle/ncbijs`](https://github.com/gagle/ncbijs) as a working
example of the matrix-publish pattern.

## Related

- [`gagle/npm-trust`](https://github.com/gagle/npm-trust) — OIDC trust
  setup, doctor, and the `setup-npm-trust` skill.
- [`gagle/npm-trust/blob/main/docs/bootstrap.md`](https://github.com/gagle/npm-trust/blob/main/docs/bootstrap.md)
  — the multi-repo architecture vision (release skill, verify skill,
  future bootstrap CLI, why this works for AI-driven solo dev).
- [`addyosmani/agent-skills`](https://github.com/addyosmani/agent-skills)
  — the canonical example of a Claude Code marketplace plugin (this
  repo's structure is modeled on it).

## Contributing

This project follows an AI-only contribution model — see
[`CONTRIBUTING.md`](./CONTRIBUTING.md). PRs are disabled. Open an issue
or discussion for change requests.

## License

MIT.
