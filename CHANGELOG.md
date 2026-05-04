# Changelog

## v0.5.3 — Auto-chain on trust gap; foolproof manual handoffs; trust-state cache

Three coordinated changes that codify the "skills are operators, not
advisors" principle and dramatically reduce per-release latency for
monorepos with many packages.

### Auto-chain on trust/init gaps

`/release` now auto-invokes `/solo-npm:trust` (or `/solo-npm:init`)
when Phase A.3's check detects a fixable gap, instead of pausing to
ask the user. The agent does NOT surface alternative invocation paths
(e.g., the `pnpm npm-trust:setup` script). One path, the skill,
executes.

- `release.md`: new "Principle: skills are operators, not advisors"
  section. Phase A.3 restructured into Cold path / Targeted path /
  Hot path (cache-aware) and Auto-fix / Hard-stop / Informational
  (issue codes). `WORKFLOWS_NONE` and "trust missing" cases now
  auto-invoke the relevant sibling skill rather than stopping with
  a "see /solo-npm:trust" suggestion.
- `trust.md`: Steps 2 (GitHub repo) and 3 (publish workflow)
  auto-confirm when unambiguous; only prompt when ≥2 plausible
  candidates remain.

Minimum-friction shape codified: **one `AskUserQuestion` per
destructive operation, max** — trust's configure gate + release's
approval gate. Everything fixable auto-chains.

### Foolproof manual handoffs (npm login + web 2FA)

The previous trust run failed because the agent tried to execute
`npm-trust --auto --only-new ...` inside its own non-interactive bash
subprocess. That command needs the npm web 2FA flow which can't be
driven from a subprocess. The agent timed out, fumbled with broken
alternative `npm trust github` invocations, and only after several
failed attempts instructed the user to run the command manually —
buried under failure noise.

`trust.md` Steps 7 and 9 now hand off to the user's terminal **early
and unambiguously** with verbatim numbered step-by-step instructions,
including:

- The exact command to run (`npm login`, then the configure command).
- What text the user will see in their terminal (verbatim code blocks).
- Each click/keystroke to perform in the browser ("Press ENTER",
  "Check the **'Skip 2FA for the next 5 minutes'** checkbox", "Click
  the **'Authenticate'** button").
- The expected end state per step.
- The exact reply the user should give back to the chat (`continue` /
  `done`).
- Notes for common failure modes (`EOTP` errors, "0 configured, N
  already set", per-package failures).

The agent **never spawns a Bash subprocess for the configure command**
— eliminating the previous failure mode entirely.

### Trust-state cache (`.solo-npm/state.json`)

For monorepos with many packages, `npm-trust --doctor` iterates ~3 npm
calls per package on every release — slow and mostly redundant once
trust is established. New cache file at `.solo-npm/state.json`
(committed; non-secret) tracks the trust-configured set with a 7-day
TTL.

Phase A.3 now branches on cache state:

| Branch | Condition | npm calls |
|---|---|---|
| **Hot path** | Cache fresh, no new packages | 0 — env-only quick checks |
| **Targeted path** | Cache fresh, new packages added | ~3 per *new* package |
| **Cold path** | Cache stale or missing | Full `--doctor` run, repopulates cache |

For ncbijs (43 packages), this drops daily release time from ~30s
trust check to ~0s. New packages added since the last release are
auto-detected and configured via `--packages <new>` targeted check;
the rest of the package set is skipped.

To force a full re-verify: delete `.solo-npm/state.json`.

- `release.md`: cache-aware Phase A.3 branching, cache update on
  Phase C.6 (CI green) success.
- `trust.md`: Step 11 (verify) updates the cache after configure.
- `init.md`: scaffolds `.solo-npm/state.json` with empty cache;
  drops the (parallel-but-redundant) `package.json#scripts.npm-trust:setup`
  scaffolding.
- `status.md`: reads the cache for the Trust column instead of
  calling `--doctor`. Surfaces stale-cache warning when applicable.

### Migration

Existing consumers (rfc-bcp47, ncbijs, etc.) should:

1. Pull `solo-npm@v0.5.3` (`/plugin marketplace update gllamas-skills`).
2. Optionally drop `package.json#scripts.npm-trust:setup` (no longer
   needed; `/solo-npm:trust` is the canonical entry point).
3. Create `.solo-npm/state.json` populated with their currently
   trust-configured packages (or run `/release` once to let the
   cold-path doctor populate it).

## v0.5.2 — Strip `name:` frontmatter from commands to match addyosmani

Even after moving entries to `.claude/commands/<name>.md` in v0.5.1,
Claude Code's autocomplete still rendered them with the skill format
(`/<name> (<plugin>)`) instead of the command format
(`/<plugin>:<name>`). Comparing our command files against
addyosmani/agent-skills's `.claude/commands/spec.md`, the difference
was in the frontmatter:

- addyosmani: `description: <one-line>` only.
- ours: `name: <name>` + `description: >` (block scalar, multi-line).

The `name:` field is for skill folders (`skills/<name>/SKILL.md` where
the directory name is the canonical name). For commands (flat .md
files), the filename IS the name and the `name:` field signals "this
is a skill" to Claude Code's parser, triggering the skill display
format even though the file lives in `.claude/commands/`.

This release strips the `name:` field from every command and inlines
the description as a one-line string, matching addyosmani's shape
exactly.

## v0.5.1 — Move skills → commands for namespaced autocomplete

Claude Code's autocomplete renders **plugin commands** (in `.claude/commands/`)
as `/<plugin>:<name>` literally in the dropdown — exactly the
`/agent-skills:spec` style of `addyosmani/agent-skills`. Plugin **skills**
(in `skills/<name>/SKILL.md`) render as `/<name> (<plugin>)` in the
dropdown — same canonical invocation, different display.

This release relocates the 7 entries from `skills/<name>/SKILL.md` to
`.claude/commands/<name>.md` so they show up as `/solo-npm:init`,
`/solo-npm:release`, etc. in the autocomplete dropdown — matching
addyosmani's DX.

- Moved `skills/init/SKILL.md` → `.claude/commands/init.md`
- Moved `skills/release/SKILL.md` → `.claude/commands/release.md`
- Moved `skills/verify/SKILL.md` → `.claude/commands/verify.md`
- Moved `skills/trust/SKILL.md` → `.claude/commands/trust.md`
- Moved `skills/status/SKILL.md` → `.claude/commands/status.md`
- Moved `skills/audit/SKILL.md` → `.claude/commands/audit.md`
- Moved `skills/deps/SKILL.md` → `.claude/commands/deps.md`
- Removed empty `skills/` directory.
- Added `"commands": "./.claude/commands"` to `.claude-plugin/plugin.json`.

The functional behavior is identical (per Claude Code docs, "commands and
skills work the same way" and "support the same frontmatter"). This is a
pure display fix.

## v0.5.0 — Complete lifecycle plugin (7 skills)

**Major restructure: solo-npm now owns the full AI-driven solo-dev npm
publishing lifecycle in seven skills, organized by phase:**

- **Bootstrap**: `init`, `trust`
- **Per-release**: `release`, `verify`
- **Portfolio operations**: `status`, `audit`, `deps`

**Breaking changes:**

- **Renamed `setup` → `trust`.** The OIDC trust skill (previously in
  the [`gagle/npm-trust`](https://github.com/gagle/npm-trust) repo) has
  moved into solo-npm and is renamed for clarity. Invocation:
  `/solo-npm:trust` (was `/npm-trust:setup`). Skill content is
  unchanged.
- **`init` rewritten as umbrella.** Phase 1 scaffolds GitHub-side files
  (release.yml + publishConfig + .nvmrc + consumer wrappers +
  `.claude/settings.json`); Phase 2 gates on first manual publish if
  package isn't on npm yet; Phase 3 chains into `/solo-npm:trust` for
  OIDC config; Phase 4 done. Idempotent — safe to re-invoke.
- **`release` and `verify` reframed as wrappable baselines.** Their
  `description` fields explicitly tell Claude to prefer wrappers when
  present. Consumer repos are expected to use thin wrappers in
  `.claude/skills/<name>/SKILL.md` to add repo-specific narrative.
  Daily DX is `/release` (wrapper) and `/verify` (wrapper); the
  marketplace baselines (`/solo-npm:release`, `/solo-npm:verify`) are
  for direct/unmodified invocation.
- **Three new skills** round out the lifecycle:
  - `/solo-npm:status` — read-only portfolio dashboard with action hints
  - `/solo-npm:audit` — security audit with risk triage (4 tiers); chains to deps for fixes
  - `/solo-npm:deps` — dep upgrade orchestrator with `/verify` gates, batched, rollback on failure, AskUserQuestion gates for major bumps
- **Marketplace.json** drops the `npm-trust` plugin entry. npm-trust
  is a pure CLI as of [`npm-trust@0.9.0`](https://github.com/gagle/npm-trust),
  not a Claude Code plugin. The `/solo-npm:trust` and `/solo-npm:audit`
  skills call it as a tool.

**Migration:**

- Switch invocations: `/npm-trust:setup` → `/solo-npm:trust`. Skill
  content is unchanged.
- If you have committed `.claude/skills/setup/SKILL.md` from the old
  `npm-trust --init-skill setup` flow, delete it; the skill now lives
  only in the marketplace plugin. Invoke as `/solo-npm:trust` directly.
- New consumers: paste the `extraKnownMarketplaces` + `enabledPlugins`
  block into `.claude/settings.json` (see README), accept the install
  prompt, run `/solo-npm:init`.
- Existing consumers with `.claude/skills/release/` or
  `.claude/skills/verify/`: replace with thin wrappers that invoke the
  marketplace baseline. See README's "Composition pattern" section.

## v0.4.0 — Plugin namespacing + drop redundant CI verification

- Renamed plugin: `release-solo-npm` → `solo-npm`. New invocations:
  `/solo-npm:release`, `/solo-npm:init`, `/solo-npm:verify`.
- Skill folders renamed: `release-solo-npm` → `release`,
  `init-solo-npm` → `init`, `verify-solo-npm` → `verify`.
- Removed Phase C.4.5 (ci.yml gating) from the release skill — for
  100% LLM-written code, local `/verify` + tag-triggered `release.yml`
  pre-publish verification is sufficient. CI was redundant.
- README + docs updated to reflect new slash invocations.
- (Brief tag-only release; later superseded by v0.5.0 lifecycle work.)

## v0.3.0 — Init skill

- Added `/solo-npm:init` skill: scaffold release.yml + publishConfig
  + .nvmrc + npm-trust:setup script for fresh repos. Public + private
  registry support.

## v0.2.0 — Marketplace plugin

- Packaged as a Claude Code marketplace plugin
  (`gagle/release-solo-npm`).
- Added `init` and `verify` skills alongside the original `release`
  skill.

## v0.1.0 — Initial release

- The `release` skill (formerly `solo-npm-release-skill`): three-phase
  tag-triggered npm release with OIDC + provenance and one
  `AskUserQuestion` approval gate.
