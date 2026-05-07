---
description: Manage npm dist-tags post-publish — add, remove, repoint, list, or cleanup stale tags across the portfolio. Triggers from prompts like "cleanup stale @next", "repoint @latest to 1.5.2 — 1.6.0 has a bug", "add @canary to 1.6.0-experimental.2 across all packages", "what dist-tags are set on my packages", "remove @next from @ncbijs/eutils". AI-driven; no manual `npm dist-tag` invocations needed.
---

# Dist-tag

Manage npm dist-tags after a version is already published — the lever for channel cleanup, rollback (`@latest` repoint after a botched release), opt-in channels (`@canary`, `@experimental`), and bulk portfolio operations.

## Phase −0 — Help mode (per `/unpublish` canonical)

If the user's prompt contains `--help` / `-h` / `"how does /solo-npm:dist-tag work"` / similar, surface a help summary **INSTEAD** of running the skill.

Synthesize from the **Operations** (add / rm / ls / repoint / cleanup-stale), Phase outline (0 / 0.5 / 0.5b / A / B / C / D), and 2–3 trigger phrases (e.g., *"cleanup stale @next"*, *"repoint @latest to 1.5.2"*). Include the HARD STOP (cannot remove `@latest`). See `/unpublish` Phase −0 for canonical format.

After surfacing, **STOP**. Re-invocation without help triggers runs normally.

## When to use

- **Stale `@next` cleanup** after a `/prerelease` PROMOTE leaves `@next` pointing at a superseded beta.
- **Rollback** after a botched release: repoint `@latest` to the previous good version while you fix-forward.
- **Opt-in channels**: publish a `@canary` or `@experimental` tag for downstream testers without disrupting `@latest` users.
- **Portfolio audit**: list current dist-tags across every package in one go.

For dist-tags at *publish time*, the `release.yml` three-layer detection step (scaffolded by `/solo-npm:init`) handles `@latest` / `@next` / `@v<major>` automatically. This skill is for *post-publish* mutations.

## Authentication note

`npm dist-tag` is **not** covered by OIDC Trusted Publishing — OIDC only covers `npm publish`. Mutating dist-tags requires a local `npm login` session. The skill's Phase A surfaces a foolproof handoff if auth is missing.

## How it works

Five operations, one skill body. Phase 0 reads the user's prompt to pick the operation and pre-fill targets.

| Operation | Effect |
|---|---|
| `add` | Point a tag at a specific version |
| `rm` | Remove a dist-tag (cannot remove `@latest`) |
| `ls` | List current dist-tags across the portfolio (read-only) |
| `repoint` | Move an existing tag to a different version (= `add` with overwrite) |
| `cleanup-stale` | Bulk: remove `@next` where it points at a pre-release whose stable equivalent has shipped to `@latest` |

## Phase 0 — Read prompt context

Scan the user's prompt for hints; pre-fill what's clear, only AskUserQuestion for what's missing.

| Prompt mentions | Pre-fill |
|---|---|
| *"cleanup stale @next"*, *"clean up stale tags"*, *"remove dead @next"* | `OPERATION = cleanup-stale`, `TAG = next` |
| *"repoint @latest to 1.5.2"*, *"point @latest at 1.5.2"*, *"rollback @latest to 1.5.2"* | `OPERATION = repoint`, `TAG = latest`, `VERSION = 1.5.2` |
| *"add @canary to 1.6.0-experimental.2"*, *"tag 1.6.0-rc.1 as canary"* | `OPERATION = add`, `TAG = canary`, `VERSION = 1.6.0-experimental.2` |
| *"remove @next from @ncbijs/eutils"*, *"drop the @experimental tag"* | `OPERATION = rm`, `TAG = next` (or `experimental`), `SCOPE = @ncbijs/eutils` |
| *"what dist-tags are set"*, *"show me the dist-tags"*, *"list dist-tags"* | `OPERATION = ls` |
| `@<scope>/<pkg>` mentioned | `SCOPE = <that-package>` (else `SCOPE = all` workspace packages) |

Examples:
- *"Cleanup stale @next across all packages"* → `OPERATION=cleanup-stale, TAG=next, SCOPE=all`. Skip all subsequent prompts; jump to Phase A.
- *"Repoint @latest to 1.5.2 on @ncbijs/eutils — the 1.6.0 release has a bug"* → `OPERATION=repoint, TAG=latest, VERSION=1.5.2, SCOPE=@ncbijs/eutils`. Skip prompts; jump to Phase A.
- Bare *"manage dist-tags"* → ask for `OPERATION` first.

If extraction is ambiguous, pre-fill what's clear and AskUserQuestion for the rest. Don't over-extract.

## Phase 0.5 — Prompt-extraction validation (E1, E3 from v0.11.0)

After Phase 0 pre-fills slots, validate against the canonical regex framework in `/unpublish` Phase 0.5. Slots specific to `/dist-tag`:

- **`TAG`** — dist-tag regex `^[a-z][a-z0-9-]*$`. Rejects "Latest" (uppercase), "@latest" (with @ prefix), "v1" (numeric leading char).
- **`VERSION`** (for `add`/`repoint`) — semver regex.
- **`SCOPE` / package name** — npm name regex.

On validation failure, STOP with diagnostic.

## Phase 0.5b — Shell-safety hardening (Tier-4 #4 from v0.13.0)

Apply the shell-safety check from `/unpublish` Phase 0.5b (canonical) to `TAG`, `VERSION`, `SCOPE`, and package name slots. Same metacharacter blacklist + double-quoted interpolation convention. Particularly important for `TAG` since it's interpolated directly into `npm dist-tag add/rm` arguments.

## Phase A — Pre-flight + state read

1. **Auth check**: run `npm whoami`. If exit code != 0 (not authenticated), surface foolproof handoff:

   > Run `npm login` to authenticate with npm:
   > 1. Type `npm login` and press Enter.
   > 2. It prints a URL — open it in your browser.
   > 3. Sign in to npmjs.com.
   > 4. Approve the login via your authenticator app (web 2FA).
   > 5. Return here when the terminal shows your username.

   Then AskUserQuestion:
   ```
   Header:   "Authenticated"
   Question: "Done with `npm login`?"
   Options:
     - Yes, continue (Recommended) — re-runs `npm whoami` to verify
     - Abort
   ```

2. **Workspace discovery** (if `SCOPE=all`): use `npm-trust --doctor --json --workflow release.yml` (cached), filter `workspace.packages[]` to `private: false`. If single-package repo, list has one entry.

3. **Per-package state read**: for each target package, fetch dist-tags:

   ```bash
   timeout 30 npm view <pkg> dist-tags --json 2>/dev/null
   ```

   Returns `{ "latest": "...", "next": "...", ... }`. The `2>/dev/null` redirect prevents npm warnings (deprecation notices, registry hints) from corrupting the JSON parse downstream — a real silent-failure mode if the user's npm config emits notices. The `timeout 30` bounds the call against a hung/slow registry.

4. **Compute proposed mutation set** based on `OPERATION`:

   - **add / repoint**: per package, `npm dist-tag add <pkg>@<VERSION> <TAG>`.
   - **rm**: per package where the tag exists.
   - **ls**: just render; no mutations.
   - **cleanup-stale**: per package, mark `@next` for removal if:
     - `next` exists in dist-tags, AND
     - `next` is pre-release-shaped (`^\d+\.\d+\.\d+-[a-z]+\.\d+$`), AND
     - `next`'s base version (stripped of `-id.n` suffix) is `<= latest` per semver.

5. **Safety rejections** (STOP before Phase B):

   | Condition | STOP message |
   |---|---|
   | `OPERATION=rm` AND `TAG=latest` | *"Cannot remove `@latest` — npm rejects this. Use `repoint` to point `@latest` at a different version first."* |
   | `OPERATION=add/repoint` AND target version doesn't exist on registry | *"Version `<VERSION>` not published for `<pkg>`. Aborting."* |
   | `OPERATION=add/repoint` AND VERSION is empty/invalid | *"Need a concrete version to add — got `<VERSION>`."* |
   | Empty mutation set after compute (e.g., cleanup-stale found nothing to clean) | *"No mutations needed — all dist-tags are current."* (graceful exit, not error) |

## Phase B — Render plan + AskUserQuestion gate

For `ls`: render the portfolio table directly; **no gate** (read-only).

```
Dist-tags across portfolio:

  @ncbijs/eutils:
    @latest = 1.6.0
    @next   = 1.6.0-beta.3   ⚠ stale (superseded by @latest)
  @ncbijs/blast:
    @latest = 1.6.0
    @next   = 1.6.0-beta.3   ⚠ stale (superseded by @latest)
  rfc-bcp47:
    @latest = 0.13.0
  npm-trust:
    @latest = 0.9.0
    @canary = 0.10.0-rc.1
```

For mutations (`add` / `rm` / `repoint` / `cleanup-stale`): render the diff verbatim, then AskUserQuestion:

```
Proposed dist-tag mutations across <N> packages:

  @ncbijs/eutils  rm @next  (was → 1.6.0-beta.3, superseded by @latest=1.6.0)
  @ncbijs/blast   rm @next  (was → 1.6.0-beta.3, superseded by @latest=1.6.0)
  ...
```

```
Header:   "Apply dist-tag changes"
Question: "Apply <N> dist-tag mutations?"
Options:
  - Proceed with <N> changes (Recommended)
  - Abort
```

## Phase C.0 — Error-handling patterns (H1, H2, H4, H5, H6, H8 from `/unpublish` reference)

**H8 rate-limit backoff (v0.12.0)**: Phase A.3 fetches dist-tags per-package via `npm view <pkg> dist-tags --json`. For portfolio-wide ops (`SCOPE=all` against many packages), wrap each `npm view` in `npm_with_h8_backoff` from `/unpublish` Phase −1.9. Same exhaustion handling as `/deprecate`.

**Carry-forward patterns:**

Before destructive `npm dist-tag` calls, apply the standard solo-npm error patterns. Canonical wording lives in `/unpublish` Phases C.0–D.2; concrete adaptation per pattern:

- **H1 — OTP / 2FA-on-writes**: detect `EOTP` / `OTP required` in `npm dist-tag` stderr. Surface manual handoff: *"npm requires an OTP. Run `npm dist-tag add <pkg>@<v> <tag> --otp=<your-OTP>` (or `npm dist-tag rm <pkg> <tag> --otp=<your-OTP>`) manually outside the skill, then re-invoke `/solo-npm:dist-tag` to resume the remaining mutations."*
- **H2 — `.solo-npm/state.json` corruption guard**: any `JSON.parse(state.json)` read (e.g., trust cache in Phase A.2 doctor invocation) must be wrapped in try/catch. On parse fail surface non-fatal warning *".solo-npm/state.json is malformed; treating as empty cache. Remove the file and re-run any solo-npm skill to regenerate."* Continue with empty defaults.
- **H4 — Registry propagation lag retry**: Phase D verify (`npm view <pkg> dist-tags --json`) uses 3 attempts × 5s sleep before declaring inconsistency. Don't HARD STOP if still inconsistent — surface non-fatal note: *"Registry not yet reflecting dist-tag mutation after 15s — npm CDN may take up to 5 minutes; re-check later with `npm view <pkg> dist-tags`."*
- **H5 — Concurrent invocation lock**: at Phase C start (per package), acquire `.solo-npm/locks/<sanitized-pkg-name>.lock` (PID file with `trap` cleanup). Refuse to start if another solo-npm skill holds it.
- **H6 — Chain-target failure recovery**: when auto-chained from `/status` (stale-`@next` warning) or `/prerelease` PROMOTE Phase E, capture any internal STOP and surface upward to the parent skill's H6 handler. Don't silently swallow.

## Phase C.0a — H3 auth-race re-check + H5 lock acquisition (concrete impl)

Mirrors `/solo-npm:deprecate` Phase C.0a (canonical wording). Phase A.1 ran `npm whoami` and stashed the result; this re-check fires immediately before the first destructive call.

```bash
# H3: re-check npm session is still valid AND still same user
WHOAMI_AT_PHASE_A="$(cat /tmp/.solo-npm-whoami-dist-tag 2>/dev/null)"
WHOAMI_NOW=$(timeout 30 npm whoami 2>/dev/null)
if [ -z "$WHOAMI_NOW" ] || [ "$WHOAMI_NOW" != "$WHOAMI_AT_PHASE_A" ]; then
  echo "ERROR: npm session expired or changed during Phase B gate."
  echo "       Phase A: $WHOAMI_AT_PHASE_A    Now: ${WHOAMI_NOW:-<no session>}"
  echo "       Re-authenticate, then re-invoke /solo-npm:dist-tag."
  exit 1
fi

# H5: per-package lock with stale-PID auto-cleanup (per /unpublish Phase −1.8)
for PKG in $TARGET_PACKAGES; do
  LOCK_FILE=".solo-npm/locks/$(echo "$PKG" | sed 's|/|_|g').lock"
  if [ -f "$LOCK_FILE" ]; then
    STALE_PID=$(cat "$LOCK_FILE" 2>/dev/null)
    if [ -n "$STALE_PID" ] && kill -0 "$STALE_PID" 2>/dev/null; then
      echo "ERROR: another solo-npm skill holds $LOCK_FILE (PID $STALE_PID alive)."
      exit 1
    fi
    [ -n "$STALE_PID" ] && echo "WARN: removing stale lockfile $LOCK_FILE (PID $STALE_PID dead)"
    rm -f "$LOCK_FILE"
  fi
  mkdir -p .solo-npm/locks
  echo $$ > "$LOCK_FILE"
done
trap 'for PKG in $TARGET_PACKAGES; do rm -f ".solo-npm/locks/$(echo "$PKG" | sed "s|/|_|g").lock"; done' EXIT
```

## Phase C — Execute

For each mutation, with **200ms inter-call backoff** to avoid registry rate limits:

```bash
# add / repoint:
npm dist-tag add <pkg>@<VERSION> <TAG>

# rm / cleanup-stale:
npm dist-tag rm <pkg> <TAG>
```

Halt on first failure. Surface npm error verbatim; do not retry automatically (auth issues, missing versions, etc. require user attention).

If a mutation fails partway through a bulk operation, surface clearly:

```
Applied 5 of 8 mutations. Halted on:
  @ncbijs/datasets: npm error E403 You cannot publish over the previously published versions: …

Retry options:
  - Resume from where we stopped (5 already done; will retry the failed mutation)
  - Abort and audit state with /solo-npm:status
```

## Phase D — Verify + summarize

For each affected package, re-fetch dist-tags via `npm view <pkg> dist-tags --json` and confirm the mutation landed.

Print final summary:

```
Applied <N> dist-tag mutations:

  @ncbijs/eutils:
    @next  removed
  @ncbijs/blast:
    @next  removed
  ...

Verified on registry. Run /solo-npm:status to refresh the dashboard.
```

## Failure modes

| Failure | Where | Recovery |
|---|---|---|
| `npm whoami` reports not authenticated | Phase A.1 | Foolproof `npm login` handoff |
| Workspace discovery returns empty | Phase A.2 | STOP — *"No publishable packages found in workspace."* |
| Cannot remove `@latest` | Phase A.5 | STOP — suggest `repoint` instead |
| Version doesn't exist for `add`/`repoint` | Phase A.5 | STOP — list available versions |
| npm rate limit (429) during Phase C | Phase C | Surface; backoff retry once; otherwise halt |
| Auth expires mid-Phase-C | Phase C | Halt; user re-authenticates and resumes |

## What this skill does NOT do

- **Does NOT touch publish-time dist-tag detection** in `release.yml`. The three-layer detection (publishConfig.tag → version-shape → default `latest`) is owned by `/solo-npm:init`; this skill is post-publish only.
- **Does NOT clean up `@latest`** automatically. `@latest` is sacrosanct; only an explicit `repoint` user action moves it.
- **Does NOT chain to `/solo-npm:deprecate`** automatically when removing tags from old versions. If you want to deprecate alongside repointing, invoke `/solo-npm:deprecate` separately (or accept the chain offer in `/release` Phase G after a major release).
- **Does NOT support OIDC-only auth** for dist-tag mutations. npm requires a logged-in session for these. If you're OIDC-only and never `npm login`-ed locally, this skill prompts you through the foolproof handoff.

## Composition

- `/solo-npm:status` stale-`@next` warning suggests `→ /solo-npm:dist-tag cleanup-stale` as the action.
- `/solo-npm:prerelease` PROMOTE Phase D offers an optional `/solo-npm:dist-tag cleanup-stale` chain after the stable version ships.
- `/solo-npm:release` rollback story: when a release goes wrong, `/solo-npm:dist-tag repoint @latest <previous-version>` is the immediate lever before fix-forward.
