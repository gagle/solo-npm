# Contributing to release-solo-npm

This project is developed and maintained entirely through AI-assisted
development using [Claude Code](https://docs.anthropic.com/en/docs/claude-code).
All changes — features, fixes, refactoring, tests, documentation — are
generated, reviewed, and verified by Claude.

**Pull requests are disabled.** Contributors cannot submit code
directly. To request a change,
[open an issue](https://github.com/gagle/release-solo-npm/issues)
or start a
[discussion](https://github.com/gagle/release-solo-npm/discussions);
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

1. **[Open an issue](https://github.com/gagle/release-solo-npm/issues)**
   for bugs, concrete features, or specific improvements — include
   expected behavior and reproduction steps where relevant.
2. **[Open a discussion](https://github.com/gagle/release-solo-npm/discussions)**
   for open-ended requests, design input, or trade-off questions.
3. The maintainer triages the request and, when accepted, implements
   it via Claude Code. The resulting commits are linked back to your
   issue or discussion.

## What this repo enforces

- **Conventional commits** (`feat:`, `fix:`, `chore:`, etc.).
- **Skill content discipline**: every change to `skills/*/SKILL.md`
  preserves the `name`/`description` frontmatter contract and the
  three-phase shape for `/release-solo-npm`.
- **Marketplace plugin compatibility**: changes to `.claude-plugin/`
  manifests are validated against the Claude Code marketplace spec.
- **AI-only contribution model**: PRs disabled at the GitHub level.

## Questions

Open a [GitHub Discussion](https://github.com/gagle/release-solo-npm/discussions)
for questions about the project, its architecture, or this
contribution model.
