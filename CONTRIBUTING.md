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
- **Visual assets are SVG only**: the README's hero banner (`resources/banner.svg`) and lifecycle diagram (`resources/lifecycle.svg`) are hand-written SVGs. No Mermaid blocks anywhere. Edit the SVG source directly; GitHub renders it via `<img>` tag. Camo cache busts on content-hash change.

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
both. After `/reload-plugins`, the seven `/solo-npm:*` invocations
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
6. Test with `claude --plugin-dir .` and verify `/solo-npm:<name>`
   appears in autocomplete.

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
