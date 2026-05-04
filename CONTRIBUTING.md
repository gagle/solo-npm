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
following the conventions in `CLAUDE.md` (when added) and
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

## Dogfooding

To test changes locally without going through the marketplace install:

```bash
cd ~/projects/solo-npm
claude --plugin-dir .
```

In that session, all `/solo-npm:*` invocations resolve to your local
source. Each new skill or change in `skills/<name>/SKILL.md` takes
effect after `/reload-plugins` (no need to restart the Claude Code
session).

The repo also keeps `.claude/skills/<name>` symlinks that target
`skills/<name>` — these expose bare invocations (`/release`,
`/verify`, etc.) when Claude Code is launched without `--plugin-dir`.
Use `--plugin-dir .` to test the namespaced (`/solo-npm:release`)
flow.

## Adding a new skill

When adding a skill to the plugin:

1. Create `skills/<name>/SKILL.md` with frontmatter `name: <name>` and
   a tight `description` (under ~200 chars; Claude truncates).
2. Add a dogfood symlink: `ln -s ../../skills/<name>
   .claude/skills/<name>`.
3. Update `.claude-plugin/plugin.json#description` to mention the new
   skill.
4. Update `marketplace.json#plugins[0].description` if it lists skill
   names by category.
5. Add a section to `README.md` under the matching lifecycle category.
6. If the skill has a daily-use cadence (like `release`/`verify`),
   consider whether it deserves a consumer wrapper template — and update
   `skills/init/SKILL.md` to scaffold it.

## Questions

Open a [GitHub Discussion](https://github.com/gagle/solo-npm/discussions)
for questions about the project, its architecture, or this
contribution model.
