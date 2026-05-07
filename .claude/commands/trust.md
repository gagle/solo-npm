---
description: Configure npm OIDC Trusted Publishing across the repo's packages — interactive wizard via the npm-trust CLI. Triggers from prompts like "configure OIDC trust", "set up Trusted Publishing", "fix CI publish failure (missing trust)", "add trust config for @scope/newpackage". Walks the user through any required manual steps (npm login, web 2FA). Use on first OIDC setup or when adding new packages.
---

# Trust

Interactive wizard for configuring npm OIDC Trusted Publishing. The
`npm-trust` CLI handles workspace detection, filtering, and per-package
configuration; this skill orchestrates the end-to-end flow:
gather inputs → confirm → handle manual steps → execute → verify.

## Phase −0 — Help mode (per `/unpublish` canonical)

If the user's prompt contains `--help` / `-h` / `"how does /solo-npm:trust work"` / similar, surface a help summary **INSTEAD** of running the skill.

Synthesize from the **Phases** (Pre-flight CLI resolution / Phase 1 discover / Phase 2 execute → auth gate → dry-run → configure → verify), and 2–3 trigger phrases (e.g., *"configure trust for my packages"*, *"setup OIDC"*). Note `/trust` orchestrates the `npm-trust` CLI (pinned to `^0.9` as of v0.16.0) — the CLI is the canonical implementation; this skill is the AI wizard wrapper. See `/unpublish` Phase −0 for canonical format.

After surfacing, **STOP**. Re-invocation without help triggers runs normally.

## When to use

- First-time OIDC trust setup for a repo's published packages.
- Incremental setup after publishing one or more new packages.
- Auditing — checking which packages are or aren't trust-configured.

## Pre-flight — resolve the CLI invocation

Different hosts reach `npm-trust` differently (source checkout, devDep,
global install, npx fetch). Decide which to use **once**, then use the same
invocation in every step below. Try in this order and stop at the first match:

1. **Source checkout.** If `./bin/npm-trust.js` exists AND
   `node -p "require('./package.json').name"` prints `npm-trust`, use:
   ```
   <CLI> = node ./bin/npm-trust.js
   ```
2. **Local devDependency.** If `./node_modules/.bin/npm-trust` is
   executable:
   ```
   <CLI> = ./node_modules/.bin/npm-trust
   ```
3. **Global install.** If `command -v npm-trust` succeeds:
   ```
   <CLI> = npm-trust
   ```
4. **Registry fetch (last resort).** Otherwise fall back to:
   ```
   <CLI> = npx -y npm-trust@^0.9
   ```
   **Pinned to major 0.9** (D3 from v0.11.0 strict-safety pass; bumped from `^0.4` in v0.16.0 after explicit verification against npm-trust 0.9.1). The skill body uses `--auto`, `--doctor`, `--json`, `--list`, `--only-new`, `--dry-run`, `--repo`, `--workflow`, `--scope`, `--packages` — all present and shape-stable in 0.9.x per the maintainer-verified compatibility check that shipped with npm-trust 0.9.1's CHANGELOG. Using `@latest` would fetch whatever is current on npm and could silently regress flag availability. Bump this pin explicitly when solo-npm is tested against a newer npm-trust major.

If your shell's `npx` form rejects the package (some npm 11 setups require
`npm exec --` instead), substitute `npm exec -- npm-trust@^0.9` for
the npx fallback.

Use the chosen invocation in **every** subsequent step. Below, `<CLI>` is the
placeholder — replace mentally.

### Verify the resolved version supports the flags this skill uses

```bash
<CLI> --help 2>/dev/null | grep -q -- "--auto" \
  && echo "ok" \
  || echo "TOO_OLD"
```

If the result is `TOO_OLD`, the resolved binary predates `--auto` (added in
v0.2.0). **Stop** and ask the user to upgrade with
`npm i -g npm-trust@^0.9` (or remove the cached version that resolved).

## Phase 1 — Discover (one call when --doctor is supported)

If the resolved CLI supports `--doctor` (added in v0.4.0), the entire Phase 1
sequence below collapses to a single call:

```bash
<CLI> --doctor --json
```

The JSON has the shape `{ schemaVersion: 1, runtime, auth, workspace, repo,
workflows, packages, issues, summary }`. Branch on:

- `summary.fail > 0` — STOP. The environment can't run the trust setup.
- `issues[]` — surface every `code` to the user, with the included `remedy`
  text where present. Notable codes: `AUTH_NOT_LOGGED_IN`,
  `WORKSPACE_NOT_DETECTED`, `WORKFLOWS_AMBIGUOUS`, `PACKAGE_NOT_PUBLISHED`,
  `PACKAGE_TRUST_DISCREPANCY`.
- `workspace.packages[]` — the detected package list.
- `repo.inferredSlug` — `<owner/repo>` for the configure call.
- `workflows[]` — candidate workflow files; if length === 1, suggest it.

If `--doctor` is not present in `<CLI> --help`, fall back to the per-step
discovery below. The skill is fully backward-compatible with v0.2.0 and
v0.3.0 CLIs.

### 1. Detect the workspace shape

```bash
<CLI> --auto --dry-run --repo placeholder/x --workflow placeholder.yml 2>&1 | head -10
```

Expected first stdout line:

- `Detected pnpm workspace — found N packages` for repos with `pnpm-workspace.yaml`.
- `Detected npm/yarn workspace — found N packages` for `package.json#workspaces`.
- `Detected single package — found 1 packages` for repos with a single root `package.json`.

If the output is `Error: --auto could not detect…`, call `AskUserQuestion`
(loading via `ToolSearch query="select:AskUserQuestion"` if needed) with:

- `header`: `"Source flag"`
- `question`: `"--auto couldn't detect the workspace. How should packages be discovered?"`
- options:
  1. `Use --scope` — Discover via npm registry under a scope (e.g. `--scope @myorg`)
  2. `Use --packages` — Explicit list of package names
  3. `Abort` — Stop the wizard

Use the chosen source flag in every subsequent step. For `Use --scope` and
`Use --packages`, follow up free-form to capture the scope or names.

### 2. Resolve the GitHub repo

```bash
git remote get-url origin
```

Parse `owner/repo` from the URL.

**Auto-confirm when unambiguous.** If the parse succeeds AND
`--doctor`'s `repo.inferredSlug` matches the parsed value, use the
slug silently — do NOT call `AskUserQuestion`. Skip to Step 3.

Only prompt via `AskUserQuestion` if:
- the parse fails or returns multiple candidates
- the doctor flags `REPO_NO_REMOTE` or `REPO_REMOTE_NOT_GITHUB`
- the parsed value disagrees with the doctor's inferred value

When prompting:

- `header`: `"GitHub repo"`
- `question`: `"Use {inferred owner/repo} as the GitHub repo for OIDC trust?"`
- options:
  1. `Confirm` — Use the inferred slug
  2. `Override` — Specify a different `owner/repo` (free-form follow-up)
  3. `Abort` — Stop the wizard

### 3. Resolve the publish workflow

If the parent skill (e.g., `/release`) passed `--workflow <name>`, use
that value silently — skip this step.

Otherwise, list candidate workflow files:

```bash
ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null
```

**Auto-pick when unambiguous.** Apply this heuristic:

1. If exactly one workflow file exists → auto-pick it.
2. If multiple files exist BUT exactly one contains `release`,
   `publish`, or `npm` in its filename → auto-pick that.
3. Otherwise (≥ 2 plausible candidates remain), call `AskUserQuestion`:
   - `header`: `"Workflow"`
   - `question`: `"Which workflow performs the npm publish?"`
   - options: one entry per candidate file (label is the basename).
     Cap at 4 options.

### 4. Show current trust state

```bash
<CLI> --auto --list
```

Per-package output: each line shows the package name and either its existing
trust configuration or `(no trust configured)`.

> **Note on cross-checking.** `npm trust list` (which the CLI calls under the
> hood) sometimes reports `(no trust configured)` even when OIDC publishing
> demonstrably works — this happens when Trusted Publishing was set up via
> npm's web UI rather than the CLI, or when the CLI's auth state is stale.
> Recent versions of `npm-trust` cross-check via `npm view <pkg>
> dist.attestations` to flag this; if you see a `(provenance present)` marker
> in the output, treat the package as effectively configured even if the
> trust line is empty.

### 5. Identify what still needs work

```bash
<CLI> --auto --only-new --list
```

The filter calls `npm trust list`, `npm view <pkg>`, and (in v0.3.0+)
`npm view <pkg> dist.attestations` per package. It keeps packages that lack
trust **and** lack provenance, or aren't yet published. The result is the
precise working set for this run.

### 6. Confirm with user

Render the summary as text (so it stays visible in the chat):

```
Detected: <source>, <N> packages
Repo:     <owner/repo>
Workflow: <file>

Already configured: <K>
Needs work:         <M>

Plan: configure OIDC trust for <M> packages.
```

Then call `AskUserQuestion` to gate Phase 2:

- `header`: `"Setup plan"`
- `question`: `"Proceed with the plan above?"`
- options:
  1. `Proceed` — Run Phase 2 (auth gate, dry-run, configure, verify)
  2. `Abort` — Stop the wizard, no state changes

## Phase 2 — Execute

### Error-handling patterns (H2, H6 from `/unpublish` reference)

- **H2 — `.solo-npm/state.json` corruption guard**: Step 11 (state cache write) reads the existing file before writing the trust update. If parse fails, surface non-fatal warning *".solo-npm/state.json is malformed; treating as empty cache."* and write a fresh structure. Don't lose the trust update on a stale cache.
- **H6 — Chain-target failure recovery**: this skill is auto-chained from `/release` Phase A.3 (delta) and `/init` Phase 3. If trust setup STOPs internally (npm-trust CLI error, web 2FA timeout, etc.), surface the verbatim diagnostic in the parent skill's context. The parent's own H6 handler will offer retry/abort.

### 7. Auth gate (hard)

Run:

```bash
npm whoami
```

**Authenticated.** Print `✓ npm whoami: <username>` and proceed to Step 8.

**Not authenticated.** Surface this **exact block verbatim** to the user, then stop and wait for the user's reply (`continue`). Do NOT proceed; do NOT try to drive `npm login` from this session.

```markdown
### npm login required

You're not logged in to npm. Please run this in your terminal (or here in chat with `!` prefix):

```
npm login
```

**What happens, step by step:**

1. The terminal prints a URL like:

   ```
   https://www.npmjs.com/login?next=/login/cli/...
   ```

   Press **ENTER** when prompted (or open the URL manually in your browser).

2. npm's login page opens in your browser. Sign in with your normal npm username and password.

3. After login, npm shows a two-factor authentication page. Enter the code from your authenticator app (or use your security key / passkey).

4. The browser shows "You are now logged in!" or similar.

5. Return to your terminal. You should see something like:

   ```
   Logged in on https://registry.npmjs.org/ as <your-username>.
   ```

6. **Reply `continue`** in this chat and I'll resume.
```

When the user replies `continue`, re-run `npm whoami`. Only proceed to Step 8 after it succeeds.

### 8. Pre-flight dry-run

Before burning a 2FA round-trip on a typo, validate the args. This step is **safe to run in agent bash** — `--dry-run` is read-only and does not require web 2FA:

```bash
<CLI> --auto --only-new --dry-run --repo <owner/repo> --workflow <file>
```

If this errors (bad repo regex, invalid workflow extension, source detection
failure), **stop and surface the error verbatim** — do not proceed to the
configure handoff.

If the dry-run output reports zero packages need configuration (e.g., all
already set), skip Step 9 entirely and proceed to Step 11 (verify) — the
configure step would be a no-op.

### 9. Configure trust — manual step required

The configure step calls `npm trust github` per package, which requires
the npm web 2FA flow. The agent **cannot drive a browser flow** from
its bash subprocess (this fails with timeouts or `EOTP` errors). Hand
off to the user's terminal with the exact block below.

Surface this **exact block verbatim** to the user, then stop and wait for the user's reply (`done`). Do NOT spawn a Bash subprocess for the configure command.

```markdown
### Configure trust — manual step required

Ready to configure trust for the **N packages** listed above. npm's web-based authentication needs your browser, which I can't drive from this session, so please run this command in your terminal (or here in chat with `!` prefix):

```
<CLI> --auto --only-new --repo <owner/repo> --workflow <file>
```

**What happens, step by step:**

1. The terminal prints something like:

   ```
   Configuring OIDC trusted publishing for N packages
   ...
   Authenticate your account at:
   https://www.npmjs.com/auth/cli/<some-uuid>
   Press ENTER to open in the browser...
   ```

2. **Press ENTER** (or open the URL manually in your browser).

3. Sign in to npm if prompted.

4. npm shows a two-factor authentication page. Look for a checkbox labelled **"Skip 2FA for the next 5 minutes"** — **check that box**.

5. Click the **"Authenticate"** button (the button text may vary slightly).

6. The browser shows "Authenticated successfully" or similar.

7. Return to your terminal. You'll see per-package progress lines:

   ```
   @scope/pkg-a   ✓ configured
   @scope/pkg-b   ✓ configured
   ...
   ```

8. Wait for the **final line**, which looks like:

   ```
   Done: <X> configured, <Y> already set, <Z> failed
   ```

9. **Reply `done`** in this chat and I'll run the verify step.

---

**Notes if something goes wrong:**

- If you see `EOTP` errors: you didn't check the "Skip 2FA for 5 minutes" box. Re-run the command and check the box this time.
- If you see `Done: 0 configured, N already set`: trust was already set up — that's fine, reply `done`.
- If any package shows `✗ failed`, copy the error and paste it here; I'll diagnose.
```

When the user replies `done`, proceed to Step 10/11.

### 10. Handle unpublished packages

If the configure output (in the user's terminal) listed failed packages with the suffix `not published yet`, surface this to the user:

> The following packages need to be published before OIDC trust can be
> configured for them:
>
> - @scope/new-pkg-a
> - @scope/new-pkg-b
>
> Publish them via your normal release process, then re-run this skill — only
> the now-published packages will be picked up.

Do not retry automatically; publishing is a deliberate user action.

### 11. Verify

This step is **safe to run in agent bash** — `--list` is read-only:

```bash
<CLI> --auto --list
```

All previously-untrusted-but-published packages should now show trust
information (the workflow file path) instead of `(no trust configured)`.

**Update the trust-state cache.** Read `.solo-npm/state.json` (create if missing using the v1 schema), then:

- Append all packages that now show trust configured to `cache.trust.configured` (deduplicate).
- Refresh `cache.trust.lastFullCheck` to `now`.
- Save.

The next `/release` (within `ttlDays`, with no new packages) takes the **hot path** — zero npm calls for the trust check.

### 12. Report

Print a final summary:

```
Setup complete.
  Configured this run:  <X>
  Already set:          <Y>
  Pending publish:      <Z>
```

If `Z > 0`, remind the user that those packages still need publishing.

## Notes

- Scope-based mode (`--scope @myorg`) is an alternative to `--auto`. Use it
  when the user explicitly wants to discover via the npm registry rather than
  the workspace filesystem — useful for cross-cutting audits of an entire org.
- `--only-new` is filesystem-scoped: it filters whichever package list was
  produced by the source flag (`--auto`, `--scope`, or `--packages`). Combine
  freely.
- The wizard never runs destructive commands; everything it executes is
  read-only or `npm trust github` (which is idempotent — re-running on an
  already-configured package returns "already configured").
- `--doctor` is the fast path for v0.4.0+ CLIs (see Phase 1 opener above).

## Recommended conventions

These are conventions the maintainer recommends for repos that adopt
`npm-trust`. They are not enforced by the CLI — adopt them for
consistency across projects.

### Use `AskUserQuestion` for every human decision

When this skill (or any skill that builds on it) needs a decision from
the user — proceed/abort, pick a workflow, override an inferred value —
the agent **must** call the `AskUserQuestion` tool. Do not write a free-
text question and stop. Free-text prompts are too easy to miss against
the surrounding agent output; the structured selector is unmissable.

### npm script naming: `npm-trust:setup`, not `setup:npm-trust`

Recommend that consumers expose the wizard under a `npm-trust:setup`
script (namespace before verb), so scripts grouped by tool sort together
and match the prevailing pnpm/monorepo convention:

```json
{
  "scripts": {
    "npm-trust:setup": "npm-trust --auto --repo <owner/repo> --workflow <file>"
  }
}
```

The older `setup:npm-trust` ordering is discouraged; rename when you
encounter it. The script invokes the `npm-trust` CLI directly (no Claude
needed), so it remains a useful shorthand even when this skill is the
preferred interactive path.
