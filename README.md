<h1 align="center">solo-npm</h1>

<p align="center"><b>The full npm publishing lifecycle for AI-driven solo developers.</b></p>

<p align="center">
Seven slash commands that take an empty repo to a tag-triggered OIDC release flow with provenance, then keep the portfolio healthy.
</p>

```
   BOOTSTRAP              ITERATE                  OPERATE
 ┌──────────────┐      ┌──────────────┐       ┌──────────────────┐
 │ /init        │ ───▶ │ /verify      │ ◀───▶ │ /status (daily)  │
 │ /trust       │      │ /release     │       │ /audit  (monthly)│
 └──────────────┘      └──────────────┘       │ /deps   (on CVE) │
   one-time             per release            └──────────────────┘
```

---

## Why solo-npm?

You're a solo developer — or running a small group of LLM agents — shipping npm packages. PRs are disabled in your repos (issue/discussion contribution model only). There's no committee, no second pair of human eyes.

Existing release tooling is built for teams: PR-based workflows, multi-stage approvals, complex changelog negotiation. In a solo or agent-driven context that overhead becomes friction — and friction makes you skip steps when you're moving fast. Skipped steps make unsigned, unverified, opaque releases.

solo-npm replaces that friction with **one structured `AskUserQuestion` checkpoint per release** and silent automation everywhere else. The skills bake in opinionated defaults — SLSA provenance attestation, OIDC Trusted Publishing, conventional-commit-driven version bumps, verify-gated dep upgrades — so you can't accidentally ship something untested or unsigned.

Beyond the release moment, the operate skills (`/status`, `/audit`, `/deps`) replace the morning ritual of opening five browser tabs to check on your portfolio. One terminal command per concern.

Tools used under the hood: [`npm-trust`](https://github.com/gagle/npm-trust) (CLI for OIDC trust config), [`gagle/prepare-dist@v1`](https://github.com/gagle/prepare-dist) (monorepo dist translation, optional).

---

## Commands

7 namespaced slash commands map to the publishing lifecycle. Each one is opinionated, idempotent, and verify-gated.

| What you're doing | Command | Key principle |
|---|---|---|
| Bootstrap a fresh repo | `/solo-npm:init` | One-shot scaffold + first-publish gate + chain into trust |
| Configure OIDC trust | `/solo-npm:trust` | `npm trust github` per package, web 2FA once |
| Run quality gates | `/solo-npm:verify` | Lint → typecheck → test → build; halt on first failure |
| Ship a release | `/solo-npm:release` | Three-phase tag-triggered publish with **one** approval gate |
| Snapshot the portfolio | `/solo-npm:status` | Read-only — version, downloads, trust, CI, drift |
| Triage security advisories | `/solo-npm:audit` | Classify CVEs into 4 actionable tiers; chain to deps |
| Upgrade dependencies | `/solo-npm:deps` | Tier-batched with `/verify` gates and rollback on failure |

---

## Tell Claude

The fastest path: open Claude Code in your repo and say:

> **Integrate solo-npm into this repo. Read https://github.com/gagle/solo-npm and follow the Quick Start.**

Claude will:

1. Fetch this README.
2. Write the marketplace block into your `.claude/settings.json`.
3. Wait for you to accept the install prompt on folder trust.
4. Run `/solo-npm:init` to scaffold release.yml + publishConfig + `.nvmrc` + consumer wrappers, then chain into `/solo-npm:trust` for OIDC.

One conversation. The repo goes from empty to ready-to-tag.

---

## Quick Start

### 1. Pin the marketplace

Create or merge into `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "gllamas-skills": {
      "source": { "source": "github", "repo": "gagle/solo-npm" }
    }
  },
  "enabledPlugins": {
    "solo-npm@gllamas-skills": true
  }
}
```

Commit it. Anyone who opens the repo gets prompted to install on first folder trust.

### 2. Accept install prompts

When you open the repo in Claude Code, accept:
- *Install marketplace `gllamas-skills`?* → Yes
- *Install plugin `solo-npm@gllamas-skills`?* → Yes

All seven `/solo-npm:*` commands resolve.

### 3. Bootstrap

```
/solo-npm:init
```

Phase 1 scaffolds:
- `.github/workflows/release.yml` (public OIDC or private token, auto-detected)
- `package.json` updates (`engines.node`, `publishConfig`, `npm-trust:setup` script)
- `.nvmrc`
- `.claude/skills/release/SKILL.md` thin wrapper (workspace-shape-aware)
- `.claude/skills/verify/SKILL.md` thin wrapper

Phase 2 gates on first manual publish (npm requires the package name to exist before OIDC trust can attach).

Phase 3 chains into `/solo-npm:trust` to configure OIDC.

That's it. From here on, daily DX is `/release`.

### Manual install (one-off testing)

```
/plugin marketplace add gagle/solo-npm
/plugin install solo-npm@gllamas-skills
```

---

## The seven commands

### Bootstrap

| Command | What it does | Use when |
|---|---|---|
| [`/solo-npm:init`](.claude/commands/init.md) | Umbrella scaffolder. Detects workspace shape (single / pnpm / Nx), generates release.yml + publishConfig + .nvmrc + consumer wrappers + settings.json. Idempotent — safe to re-invoke. Phase 2 gates on first publish. Phase 3 chains into `/solo-npm:trust`. | Fresh repo (or new package added to an existing monorepo) |
| [`/solo-npm:trust`](.claude/commands/trust.md) | Interactive OIDC wizard. Walks `npm login`, web 2FA once, runs `npm trust github` per package, verifies. Uses the [`npm-trust`](https://github.com/gagle/npm-trust) CLI. | First OIDC setup or after publishing a new package |

### Per release

| Command | What it does | Use when |
|---|---|---|
| [`/solo-npm:release`](.claude/commands/release.md) | Three phases. **A. Pre-flight** runs `/verify` + `npm-trust --doctor`, silent if green. **B. Plan** detects bump from conventional commits, renders summary, asks **one** structured `AskUserQuestion`. **C. Execute** bumps version, commits, pushes, tags, watches CI, verifies registry attestation. | Shipping any release |
| [`/solo-npm:verify`](.claude/commands/verify.md) | Lint + typecheck + test + build, halt on first failure. Auto-detects package manager and scripts. Composes with `/release` (Phase A.2 + C.4) and `/deps` (after each upgrade batch). | Pre-commit, pre-release, after deps changes |

### Operate the portfolio

| Command | What it does | Use when |
|---|---|---|
| [`/solo-npm:status`](.claude/commands/status.md) | One-page dashboard: latest version, last publish, weekly downloads, OIDC trust state, recent commits, open issues, CI health. Read-only — no state changes. Surfaces action hints (e.g., "@pkg has 2 unreleased commits → /release ready?"). | Daily/weekly portfolio check |
| [`/solo-npm:audit`](.claude/commands/audit.md) | Runs `pnpm audit`, classifies advisories into 4 tiers (Fix today / Plan upgrade / Lower priority / Note). Surfaces only the actionable subset. Read-only; chains to `/deps` for fixes. | Monthly maintenance or when a CVE lands |
| [`/solo-npm:deps`](.claude/commands/deps.md) | Tier-batched upgrade orchestrator. Classifies into trivial/safe/major/CVE-driven, applies in dep-graph order, runs `/verify` after each batch, rolls back on failure with an `AskUserQuestion` gate. Major upgrades NEVER auto-applied. | Monthly maintenance or to fix CVEs from `/audit` |

---

## Composition pattern

Marketplace commands are **opinionated baselines**. Consumer repos keep **thin wrappers** in `.claude/skills/<name>/SKILL.md` that invoke the baseline + add repo-specific narrative. The wrapper pattern preserves per-repo customization without forking the whole skill.

Daily-use commands get wrappers; one-off and read-only commands don't:

| Command | Wrapper? | Why |
|---|---|---|
| `release`, `verify` | YES | Per-repo narrative: workspace shape, prepare-dist usage, custom verification commands |
| `init`, `trust`, `status`, `audit`, `deps` | NO | One-off or auto-detected — no per-repo customization |

`/solo-npm:init` scaffolds the wrappers automatically when you bootstrap.

### What a wrapper looks like

`.claude/skills/release/SKILL.md` in a consumer repo:

```yaml
---
name: release
description: Release ncbijs packages — wraps /solo-npm:release with monorepo specifics.
---

# Release (ncbijs)

Composes /solo-npm:release with this repo's specifics.

## Repo context

- Workspace: pnpm + Nx monorepo at `packages/*`
- Versioning: unified — single git tag bumps every package
- Publish: per-package matrix in `release.yml`; uses `gagle/prepare-dist@v1`
- Repo slug: `gagle/ncbijs`
- Workflow: `release.yml`
- Verification: `/verify` runs `pnpm nx run-many -t lint typecheck build test`

## Workflow

Invoke `/solo-npm:release` for the opinionated three-phase baseline.

## Deviations from the baseline

(none today)
```

When you type `/release`, Claude Code loads the wrapper. The wrapper invokes `/solo-npm:release`. Both bodies sit in context — Claude reasons over the merged guidance (your overrides + the opinionated baseline).

---

## How it works

```
┌────────────────────────────────────────────────────────────────┐
│  solo-npm plugin (.claude/commands/<name>.md)                  │
│    Opinionated baseline workflow                               │
│    Auto-detects: workspace shape, repo slug, package manager   │
│    Composes with: npm-trust CLI, /verify, /deps                │
└──────────────────────────┬─────────────────────────────────────┘
                           │ invoked from
                           ▼
┌────────────────────────────────────────────────────────────────┐
│  Consumer wrapper (.claude/skills/<name>/SKILL.md)             │
│    Repo-specific narrative + step overrides                    │
│    Bare invocation (e.g., /release)                            │
└────────────────────────────────────────────────────────────────┘
```

**Three-phase release anatomy** (the most-used command):

| Phase | Purpose | Human input |
|---|---|---|
| **A. Pre-flight** | `/verify` + `npm-trust --doctor`. Silent if green. Halts on failure. | None |
| **B. Plan** | Detect version bump from conventional commits, render summary, render CHANGELOG draft, ask **one** structured `AskUserQuestion`. | One click: Proceed / Override / Edit changelog / Abort |
| **C. Execute** | Bump version, commit, push, tag, watch CI, verify provenance attestation on registry. | None |

One human checkpoint. Everything else is silent if green.

---

## Advanced

<details>
<summary><b>Pre-release versions</b></summary>

The release command detects `x.y.z-pre.n` shape automatically. Phase B's `AskUserQuestion` adapts:

- `Bump pre-release counter` (`1.3.0-beta.1` → `1.3.0-beta.2`)
- `Promote to stable` (`1.3.0-beta.n` → `1.3.0`)
- `Override` / `Abort`

To start a pre-release line from stable, pick `Override version` and type `1.3.0-beta.1` — accepted verbatim.

</details>

<details>
<summary><b>Custom / private registries (Verdaccio, Artifactory, GitHub Packages, ...)</b></summary>

The release command works against public npm (default) and any custom / private registry that respects npm token auth.

| Concern | Public npm | Custom / private |
|---|---|---|
| Auth in CI | OIDC (no secrets) | `NODE_AUTH_TOKEN` env from GitHub Actions secret |
| `release.yml` permissions | `id-token: write` | (no `id-token`) |
| `package.json#publishConfig` | `{ access, provenance: true }` | `{ access, registry: "<URL>" }` (no provenance) |
| SLSA provenance | ✓ | ✗ (Sigstore is npmjs.com-only) |
| `/solo-npm:trust` | configures OIDC | not applicable (token auth) |

`/solo-npm:init` detects whether `publishConfig.registry` (or `.npmrc`) already points at a custom registry and picks the right `release.yml` template + omits `provenance: true`.

NEVER commit a literal `_authToken=...` line in `.npmrc`. The token belongs in env / repo secrets only. `npm-trust --doctor` flags this as `NPMRC_LITERAL_TOKEN`.

</details>

<details>
<summary><b>Monorepos (pnpm + Nx, matrix publish, prepare-dist)</b></summary>

The release command auto-detects pnpm + Nx monorepos and iterates `packages/*` for the version bump + per-package registry verification. The init command scaffolds a monorepo-flavored release wrapper template.

See [`gagle/ncbijs`](https://github.com/gagle/ncbijs) as a working example.

**Publishing from `dist/` with `gagle/prepare-dist@v1`:** monorepos that publish from `<package>/dist/` (rather than `<package>/`) commonly pair with the [`gagle/prepare-dist@v1`](https://github.com/gagle/prepare-dist) GitHub Action — it cleans up `<dist>/package.json` (strips `dist/` prefix from paths, drops dev fields, copies README + LICENSE) and verifies the version matches the pushed tag.

| Layer | Owner |
|---|---|
| Bump source `package.json#version` | release command (Phase C.1) |
| Tag + push tag | release command (Phase C.5) |
| Translate source → `dist/package.json` | `prepare-dist` action in `release.yml` |
| `npm publish` from `<package>/dist/` | `release.yml` publish step |
| Registry verify | release command (Phase C.7) |

The consumer's release wrapper notes "uses `gagle/prepare-dist@v1`" so the agent expects the dist-translation phase in CI.

</details>

<details>
<summary><b>Dogfooding (working on solo-npm itself)</b></summary>

To test changes locally without going through the marketplace install, launch Claude Code with `--plugin-dir`:

```bash
cd ~/projects/solo-npm
claude --plugin-dir .
```

Inside that session, all commands load with the `solo-npm:` namespace. Each change to `.claude/commands/<name>.md` takes effect after `/reload-plugins`.

The plugin entries live in `.claude/commands/<name>.md` (flat markdown), not `skills/<name>/SKILL.md` (folder-per-skill). Both forms work the same way functionally, but **plugin commands** display as `/<plugin>:<name>` literally in the autocomplete dropdown while **plugin skills** display as `/<name> (<plugin>)`. We use the commands form to match `addyosmani/agent-skills`'s DX.

See [`CONTRIBUTING.md`](./CONTRIBUTING.md#dogfooding) for details.

</details>

---

## See also

- [`gagle/npm-trust`](https://github.com/gagle/npm-trust) — pure CLI for npm OIDC Trusted Publishing. The `/solo-npm:trust` and `/solo-npm:audit` commands orchestrate this CLI.
- [`gagle/prepare-dist@v1`](https://github.com/gagle/prepare-dist) — GitHub Action for translating monorepo source `package.json` → `dist/package.json` at publish time.

---

## Contributing

This project follows an AI-only contribution model — see [`CONTRIBUTING.md`](./CONTRIBUTING.md). PRs are disabled. Open an issue or discussion for change requests.
