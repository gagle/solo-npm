# Contributing to solo-npm

This project is developed and maintained entirely through AI-assisted
development using [Claude Code](https://docs.anthropic.com/en/docs/claude-code).
All changes — features, fixes, refactoring, tests, documentation — are
generated, reviewed, and verified by Claude.

**Pull requests are disabled.** Contributors cannot submit code
directly. To request a change,
[open an issue](https://github.com/gagle/solo-npm/issues)
or start a
[discussion](https://github.com/gagle/solo-npm/discussions);
the maintainer picks it up and implements it through Claude Code
following the conventions in [`CLAUDE.md`](./CLAUDE.md) and
`.claude/`.

## Why

This is an intentional design decision, not a gimmick. AI-generated
content with strict conventions, full test coverage (where applicable),
and automated review produces more consistent, maintainable output
than ad-hoc human contributions. Every change follows the same process,
every time.

## How to request a change

1. **[Open an issue](https://github.com/gagle/solo-npm/issues)**
   for bugs, concrete features, or specific improvements — include
   expected behavior and reproduction steps where relevant.
2. **[Open a discussion](https://github.com/gagle/solo-npm/discussions)**
   for open-ended requests, design input, or trade-off questions.
3. The maintainer triages the request and, when accepted, implements
   it via Claude Code. The resulting commits are linked back to your
   issue or discussion.

## What this repo enforces

- **Conventional commits** (`feat:`, `fix:`, `chore:`, etc.).
- **Skill content discipline**: every change to `skills/<name>/SKILL.md`
  preserves the `name`/`description` frontmatter contract.
- **Composition pattern**: `release` and `verify` are designed to be
  wrapped per-repo. Their descriptions tell Claude to prefer the
  consumer's `.claude/skills/<name>/SKILL.md` wrapper when present.
  Tighten descriptions deliberately; loose ones cause auto-invocation
  ambiguity.
- **Marketplace plugin compatibility**: changes to `.claude-plugin/`
  manifests are validated against the Claude Code marketplace spec.
- **AI-only contribution model**: PRs disabled at the GitHub level.
- **Visual assets are SVG only**: the README's hero banner (`resources/banner.svg`) and lifecycle diagram (`resources/lifecycle.svg`) are hand-written SVGs. No Mermaid blocks anywhere. Edit the SVG source directly; GitHub renders it via `<img>` tag. Camo cache busts on content-hash change.
- **Phase naming convention**: skills with strict-sequential phases use **numbers** (`Phase 1`, `Phase 2`, …) or **steps** (`Step 1`, …). Skills with branching or named-phase patterns use **letters with `Phase 0` reserved for prompt-context reading** (`Phase 0`, `Phase A`, `Phase B`, …, with sub-phases like `F.5` allowed). Both conventions are valid; choose based on whether the skill has branching or sub-phases. Don't mix within a single skill.
- **De-hardcode counts**: avoid hardcoding the number of skills in user-facing copy (README, banner, plugin/marketplace descriptions). Use list-form descriptions ("publish, version, dist-tag, …") instead of count-form ("twelve skills"). Self-correcting copy is cheaper than count-tracking copy.

## Dogfooding

To test changes locally without going through the marketplace install,
launch Claude Code with `--plugin-dir` pointing at this repo:

```bash
cd ~/projects/solo-npm
claude --plugin-dir .
```

In that session, all skills load **as plugin skills with the
`solo-npm:` namespace** — so `/help` shows `/solo-npm:init`,
`/solo-npm:release`, `/solo-npm:status`, etc. Each change to
`skills/<name>/SKILL.md` takes effect after `/reload-plugins` (no need
to restart the session).

> **Why `--plugin-dir .` and not `.claude/skills/` symlinks?**
> Project-local skills (`.claude/skills/<name>/SKILL.md`) load **without**
> the plugin namespace prefix — they'd show as bare `/release`, `/deps`,
> etc. We want the namespaced invocation in dogfood sessions to match
> what end users see, so this repo deliberately has no `.claude/`
> directory. The `--plugin-dir .` flag is the canonical way to load a
> plugin source as a plugin during development.

To verify the install path that consumers will use, paste this snippet
into a throwaway repo's `.claude/settings.json` and trust the folder:

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

Claude Code will prompt to install the marketplace + plugin; accept
both. After `/reload-plugins`, the `/solo-npm:*` invocations
resolve.

## Adding a new command

When adding a command to the plugin:

1. Create `.claude/commands/<name>.md` with frontmatter `description:
   <one-liner>` (under ~200 chars; Claude truncates). The filename is
   the command name; `name:` field is optional and defaults to the
   filename.
2. Update `.claude-plugin/plugin.json#description` to mention the new
   command.
3. Update `marketplace.json#plugins[0].description` if it lists
   commands by category.
4. Add a section to `README.md` under the matching lifecycle category.
5. If the command has a daily-use cadence (like `release`/`verify`),
   consider whether it deserves a consumer wrapper template — and
   update `.claude/commands/init.md` to scaffold it.
6. **Apply the patterns + conventions** in the section below — phase
   structure, error-handling references, prompt validation, etc. New
   skills follow the patterns established in v0.10.0–v0.13.0.
7. Test with `claude --plugin-dir .` and verify `/solo-npm:<name>`
   appears in autocomplete.

## Patterns + conventions (v0.10.0–v0.13.0)

Skills added or modified from v0.10.0 onward follow strict conventions
for safety, error handling, and reliability. Most patterns are
documented **canonically in `/unpublish`** (the most mature skill body)
and **referenced by name** from sibling skills — to avoid prose
duplication and keep all implementations in sync.

When adding a new pattern that applies universally, document it
canonically in `/unpublish` and brief-reference from each affected
sibling. When adding a skill-specific pattern, just document inline.

### Phase numbering — `−1` / `0` / `0.5` / `0.5b` / `A`–`G`

Skills run phases in this order:

- **Phase −1** (universal pre-flight, canonical in `/unpublish`): tool
  detection, timeout wrappers, `.gitconfig` user check, env-var
  validation, atomic state.json writes, defensive npm parsing,
  detached-HEAD / worktree / submodule detection, gh pre-flight, lock
  acquisition, rate-limit + SSL + SIGINT handlers. Sibling skills
  reference the relevant subset.
- **Phase 0**: prompt-context extraction. Pre-fill what's clear from the
  user's prompt; AskUserQuestion only for what's missing. Don't
  over-extract.
- **Phase 0.5**: regex validation of extracted slots (semver,
  identifier, scope/name) — STOP on malformed input rather than letting
  it flow into Phase A.
- **Phase 0.5b**: shell-safety hardening — reject shell metacharacters
  (`;`, `&`, `|`, backtick, `$`, parens, redirects, control chars) in
  any extracted slot that flows into shell interpolation.
- **Phase A**: pre-flight + state read. Auth check, working-tree clean,
  workspace discovery, cache reads.
- **Phase B**: render plan + AskUserQuestion gate (single question per
  gate; "Recommended" marker on default; "Abort" always available).
- **Phase C**: execute. Auth re-check (H3), lock acquisition (H5),
  destructive operations.
- **Phase D**: verify + summarize. Post-mutation `npm view` with
  propagation-lag retry (H4); state.json cache cleanup; final
  summary.
- **Phase E–G** (skill-specific): cleanup chains, post-major
  deprecation, forward-port, etc.

Don't mix the two phase-naming conventions within a skill (numbered
`Phase 1, 2, …` vs lettered `Phase A, B, …`). Choose based on whether
the skill has branching or sub-phases.

### H1–H8 error-handling pattern catalog

Eight reusable error-handling patterns established v0.10.0–v0.13.0,
canonical in `/unpublish`. When adding error handling to a new skill,
reference these by name:

| Pattern | What it handles |
|---|---|
| **H1** | OTP / 2FA-on-writes detection — surface manual handoff with `--otp=<code>` |
| **H2** | `.solo-npm/state.json` corruption guard + atomic writes |
| **H3** | Auth-window race — re-check `npm whoami` before destructive call |
| **H4** | Registry propagation lag retry — 3 attempts × 5s on post-mutation `npm view` |
| **H5** | Concurrent invocation lock — `.solo-npm/locks/<sanitized>.lock` with stale-PID auto-cleanup |
| **H6** | Chain-target failure recovery — capture child STOP messages and surface in parent context |
| **H7** | Universal tool reliability — Phase −1 pre-flight checklist |
| **H8** | Rate-limit backoff — 429 detection + exponential 1s/2s/4s/8s with jitter; max 4 attempts |

When you add a new pattern that applies universally, propose H9, H10,
etc., and document it canonically in `/unpublish`.

### npm CLI BCL convention

Every `npm <subcmd>` call solo-npm makes has known cross-version
variations (npm 6/7/8/9/10 schema differences,
private-registry edge cases, etc.). The comprehensive BCL table in
`/unpublish` Phase −1.4b enumerates ~30 patterns + the defensive-parse
fallback per call.

**At every npm call site**:

1. Wrap with `timeout 30` (or longer for CI watch).
2. Redirect stderr `2>/dev/null` if piping to a JSON parser.
3. Check return code; on failure classify per H1 / H4 / SSL / general.
4. On success, validate expected fields exist before reading; fall
   back per the BCL table.
5. For mutation calls, apply H1 + H3 + H5 + H8 wrappers.

### State.json conventions

- **Read-modify-write that preserves unknown fields** (H2) — `state.X =
  newValue` would silently drop future-schema additions; instead read
  the entire object, mutate only the relevant keys, write back the
  entire object.
- **Atomic writes** — write to `.solo-npm/state.json.tmp`, then
  `fs.renameSync()`. Single atomic inode swap on POSIX. A killed-mid-
  write process leaves the original cache intact.
- **Corruption guard on reads** — every `JSON.parse` wrapped in
  try/catch; on parse fail surface non-fatal warning, treat as empty
  cache, continue.

### Lock file convention

- Path: `.solo-npm/locks/<sanitized-pkg-name>.lock` (replace `/` with
  `_`). For per-repo locks (e.g., `/release`), use the repo root hash.
- Content: PID of the holding process.
- Acquisition: check existence; if alive (`kill -0 $PID` succeeds),
  STOP. If dead, WARN + clean + proceed.
- Cleanup: `trap 'rm -f $LOCK' EXIT` for graceful exit; SIGINT trap
  (Phase −1.11) for Ctrl+C. SIGKILL leaks the lockfile — handled by
  the next acquisition's stale-PID auto-cleanup.

### AskUserQuestion conventions

- **One question per gate**. Don't cascade multiple questions in
  sequence without intermediate state.
- **"Recommended" marker on the default option** — explicit, not
  implicit by ordering. Helps Claude pick correctly when the user is
  vague.
- **"Abort" always available** as the last option. Lets the user back
  out at any gate.
- **Header naming**: short, action-noun (`"Apply deprecation"`,
  `"Confirm unpublish"`). Not a full sentence.
- **AskUserQuestion is for human decisions**, not yes/no plumbing.
  Don't use free-text "yes/no" prompts — AskUserQuestion or nothing.

### Where to document new patterns

| Pattern type | Where to document |
|---|---|
| Universal (applies to ≥3 skills) | Canonical in `/unpublish` Phase −1 (numbered subsection); brief-reference from each affected sibling |
| Error-handling pattern (extends H1–H8) | Add H9/H10/… in the catalog above + canonical implementation in `/unpublish` |
| Skill-specific (1 skill only) | Inline in that skill's body; no need to centralize |
| Phase-structure addition (new Phase X) | Document in `/unpublish` first as the canonical reference, then propagate |

When you propagate a pattern from `/unpublish` to a sibling, the
sibling gets a brief 1-paragraph reference, NOT a duplicated
implementation. The sibling reads "see `/unpublish` Phase −1.X for
canonical wording" and applies the pattern at its own Phase A or
Phase C.

### Adding a new error-handling test

Each new error-handling pattern should land with a regression scenario
in `docs/regression.md`. Number sequentially (S34, S35, …). Each
scenario has Setup / Trigger / Expected / Verify sections per the
existing template.

> **Why `.claude/commands/<name>.md` and not `skills/<name>/SKILL.md`?**
> Both forms work the same way functionally per the Claude Code docs.
> The difference is autocomplete display: plugin commands render as
> `/<plugin>:<name>` literally in the dropdown, while plugin skills
> render as `/<name> (<plugin>)`. We chose commands to match
> `addyosmani/agent-skills`'s DX.

## Questions

Open a [GitHub Discussion](https://github.com/gagle/solo-npm/discussions)
for questions about the project, its architecture, or this
contribution model.
