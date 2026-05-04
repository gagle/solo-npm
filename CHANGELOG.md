# Changelog

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
