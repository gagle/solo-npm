---
description: Mark npm package versions as deprecated (or undeprecate) with a custom message — single version, range, or mass-deprecate across the portfolio. Triggers from prompts like "deprecate all 1.x with message 'v1.x is EOL — migrate to v2'", "mark 1.6.0 as do-not-use because of data bug", "deprecate <2.0.0 across all packages", "undeprecate 1.5.0 of @ncbijs/eutils". AI-driven; rejects unbounded ranges for safety.
---

# Deprecate

Retire a published version (or version range) cleanly with a `npm WARN` message that surfaces during `npm install`. Reversible. Safer than `npm unpublish` (which has a 24-hour hard window and breaks consumer lockfiles).

## When to use

- **Post-major-release EOL**: just shipped `2.0.0`; mark all `1.x` deprecated with a "migrate to v2" message.
- **CVE response**: deprecate the affected version range, pointing users to the fixed version.
- **Bad release recovery**: deprecate one buggy version while you fix-forward (companion to `/solo-npm:dist-tag repoint @latest`).
- **Mass undeprecate**: lift a previously-applied deprecation (set message to `""`).

## Authentication note

`npm deprecate` is **not** covered by OIDC Trusted Publishing. It requires a local `npm login` session. The skill's Phase A surfaces a foolproof handoff if auth is missing (matches the `/solo-npm:trust` and `/solo-npm:dist-tag` pattern).

## How it works

Two operations: **deprecate** (apply a message) and **undeprecate** (lift it). Phase 0 reads the user's prompt to pre-fill the operation, range, and message.

## Phase 0 — Read prompt context

| Prompt mentions | Pre-fill |
|---|---|
| *"deprecate"*, *"mark as deprecated"*, *"retire <version>"*, *"EOL"* | `OPERATION = deprecate` |
| *"undeprecate"*, *"lift deprecation"*, *"un-retire"* | `OPERATION = undeprecate` |
| Specific version: `1.6.0`, `v2.0.0-beta.1` | `RANGE = <that-version>` |
| Major: `1.x`, `v1`, `the v1 line` | `RANGE = 1.x` (semver: `>=1.0.0 <2.0.0`) |
| Comparator: `<2.0.0`, `<=1.5.0`, `>=1.0.0 <1.5.0` | `RANGE = <as-stated>` |
| The user's message in quotes or after "with message" / "saying" | `MESSAGE = <verbatim user text>` |
| Reason as a phrase: "because of data bug", "due to CVE-2026-..." | `MESSAGE` derived: *"Do not use — <reason>; upgrade to <next-version>."* |
| Single package mentioned (`@ncbijs/eutils`) | `SCOPE = @ncbijs/eutils` |
| *"all packages"*, *"across the portfolio"*, *"every package"*, or no scope | `SCOPE = all` |

Examples:
- *"Deprecate all 1.x with message 'v1.x is EOL — migrate to v2.0.0+'"* → `OPERATION=deprecate, RANGE=1.x, SCOPE=all, MESSAGE='v1.x is EOL — migrate to v2.0.0+'`. Skip all prompts; jump to Phase A.
- *"Mark 1.6.0 as do-not-use because of a data bug; users should upgrade to 1.6.1"* → `OPERATION=deprecate, RANGE=1.6.0, MESSAGE='Do not use — data corruption bug; upgrade to 1.6.1.'`. Skip all prompts.
- *"Undeprecate 1.5.0 of @ncbijs/eutils"* → `OPERATION=undeprecate, RANGE=1.5.0, SCOPE=@ncbijs/eutils, MESSAGE=""` (always empty for undeprecate).
- Bare *"deprecate the old major"* → ask for `RANGE` (which major?), then `MESSAGE`.

If extraction is ambiguous, pre-fill what's clear and ask the rest.

## Phase A — Pre-flight + state read

1. **Auth check** (same handoff as `/solo-npm:dist-tag`): `npm whoami`. If not authenticated, surface foolproof `npm login` instructions + AskUserQuestion gate.

2. **Workspace discovery** (if `SCOPE=all`): same as other skills.

3. **Per-package version enumeration**: for each target package, list versions matching `RANGE`:

   ```bash
   timeout 30 npm view <pkg> versions --json 2>/dev/null
   ```

   Filter via `semver.satisfies()` against `RANGE`. The `2>/dev/null` redirect prevents npm warnings from corrupting the JSON parse; `timeout 30` bounds against a hung registry.

4. **Per-version deprecation status read**: for each matching version:

   ```bash
   npm view "<pkg>@<v>" deprecated
   ```

   Returns the current deprecation message string, or empty if not deprecated.

   - For `OPERATION=deprecate`: skip versions whose current message is already exactly the proposed `MESSAGE`. List them in the plan summary as "already deprecated (skipped)".
   - For `OPERATION=undeprecate`: skip versions that aren't currently deprecated.

5. **Safety rejections** (STOP before Phase B):

   | Condition | STOP message |
   |---|---|
   | `RANGE` is `*`, empty, `x`, or otherwise unbounded | *"Refusing to deprecate an unbounded range. Provide a concrete range like `1.x`, `<2.0.0`, or a specific version."* |
   | `OPERATION=deprecate` AND `MESSAGE` is empty | *"Deprecation message cannot be empty (that's the undeprecate path). Provide a message like 'v1.x is EOL — migrate to v2'."* |
   | `MESSAGE` is longer than 1024 characters (npm's effective limit) | Surface warning + offer truncation: *"Message is <N> characters; npm clips at ~1024. Truncate to first 1024? Yes / No, edit / Abort"* |
   | No matching versions found in any target package | STOP gracefully: *"No versions match `<RANGE>` in any package. Nothing to do."* |
   | All matching versions are already in target state (already-deprecated for deprecate; not-deprecated for undeprecate) | STOP gracefully: *"All matching versions are already in the target state. Nothing to do."* |

## Phase B — Render plan + AskUserQuestion gate

Render the affected version list with the message:

```
Deprecate <N> versions across <M> packages with message:

    "v1.x is EOL — migrate to v2.0.0+. See migration guide: https://example.com/v2"

Affected versions:

  @ncbijs/eutils  (7 versions): 1.0.0, 1.0.1, 1.1.0, 1.1.1, 1.2.0, 1.5.0, 1.5.2
  @ncbijs/blast   (7 versions): 1.0.0, 1.0.1, 1.1.0, 1.1.1, 1.2.0, 1.5.0, 1.5.2
  ...

Already deprecated with the same message (skipped):
  @ncbijs/eutils@0.9.0
```

Then:

```
Header:   "Apply deprecation"
Question: "Deprecate <N> versions across <M> packages?"
Options:
  - Proceed with <N> deprecations (Recommended)
  - Abort
```

For undeprecate:

```
Lift deprecation on <N> versions across <M> packages?
  @ncbijs/eutils  (3 versions): 1.5.0, 1.5.1, 1.5.2

Header:   "Apply undeprecation"
Question: "Undeprecate <N> versions across <M> packages?"
Options:
  - Proceed with <N> undeprecations (Recommended)
  - Abort
```

## Phase C.0 — Error-handling patterns (H1, H2, H4, H5, H6 from `/unpublish` reference)

Before destructive `npm deprecate` calls, apply the standard solo-npm error patterns. Canonical wording lives in `/unpublish` Phases C.0–D.2; concrete adaptation per pattern:

- **H1 — OTP / 2FA-on-writes**: detect `EOTP` / `OTP required` in `npm deprecate` stderr. Surface manual handoff: *"npm requires an OTP. Run `npm deprecate <pkg>@<v> '<msg>' --otp=<your-OTP>` manually outside the skill, then re-invoke `/solo-npm:deprecate` to resume the remaining versions."*
- **H2 — `.solo-npm/state.json` corruption guard**: any `JSON.parse(state.json)` read (e.g., trust/audit cache reads in Phase A) must be wrapped in try/catch. On parse fail surface non-fatal warning *".solo-npm/state.json is malformed; treating as empty cache. Remove the file and re-run any solo-npm skill to regenerate."* Continue with empty defaults; don't crash.
- **H4 — Registry propagation lag retry**: Phase D verify (`npm view <pkg>@<v> deprecated`) uses 3 attempts × 5s sleep before declaring inconsistency. Don't HARD STOP if still inconsistent — surface non-fatal note: *"Registry not yet reflecting deprecation after 15s — npm CDN may take up to 5 minutes; re-check later with `npm view <pkg>@<v> deprecated`."*
- **H5 — Concurrent invocation lock**: at Phase C start (per package), acquire `.solo-npm/locks/<sanitized-pkg-name>.lock` with `mkdir -p .solo-npm/locks && [ -f LOCK ] && kill -0 $(cat LOCK) 2>/dev/null && exit 1; echo $$ > LOCK; trap 'rm -f LOCK' EXIT`. Refuse to start if another solo-npm skill holds it; user must wait or `rm` the stale lockfile.
- **H6 — Chain-target failure recovery**: when this skill is auto-chained from `/release` Phase G, `/audit` Phase 5, or `/unpublish` Gate 1, capture any internal STOP and surface the verbatim diagnostic upward. The parent skill's own H6 handler will offer retry/abort options. Don't silently swallow.

## Phase C — Execute

For each version of each package, with **200ms inter-call backoff**:

```bash
# Deprecate:
npm deprecate "<pkg>@<version>" "<MESSAGE>"

# Undeprecate (empty string, exactly):
npm deprecate "<pkg>@<version>" ""
```

Halt on first failure. Surface npm error verbatim. Mid-run failure summary:

```
Applied 14 of 18 deprecations. Halted on:
  @ncbijs/datasets@1.5.0: npm error E403 …

Resume options:
  - Resume from where we stopped
  - Abort and audit state with /solo-npm:status
```

## Phase D — Verify + summarize

Re-read `npm view "<pkg>@<v>" deprecated` for each affected version; confirm the message landed.

Final summary:

```
Deprecated <N> versions with message:
    "v1.x is EOL — migrate to v2.0.0+"

  @ncbijs/eutils (7 versions): 1.0.0, ..., 1.5.2
  @ncbijs/blast  (7 versions): 1.0.0, ..., 1.5.2
  ...

Users running `npm install <pkg>@<version>` will now see the deprecation message
as a `npm WARN` line.
```

## Failure modes

| Failure | Where | Recovery |
|---|---|---|
| Not authenticated | Phase A.1 | Foolproof `npm login` handoff |
| Unbounded range (`*`, empty) | Phase A.5 | STOP — require concrete bound |
| Empty message on deprecate | Phase A.5 | STOP — that's the undeprecate path |
| Message too long | Phase A.5 | Offer truncation |
| No matching versions | Phase A.5 | STOP gracefully (informational) |
| npm rate limit (429) | Phase C | Backoff + retry once; otherwise halt |
| Auth expires mid-Phase-C | Phase C | Halt; user re-authenticates and resumes |

## What this skill does NOT do

- **Does NOT `npm unpublish`**. Unpublish has a 24-hour hard window for popular packages and breaks consumer lockfiles. Deprecation is the gentle alternative; this skill is the gentle path.
- **Does NOT auto-bump versions**. If you need a fixed version, ship `/solo-npm:hotfix` or `/solo-npm:release` first, *then* invoke `/solo-npm:deprecate` to retire the bad version.
- **Does NOT touch dist-tags**. If you need to repoint `@latest` after deprecating its target version, run `/solo-npm:dist-tag repoint` separately.
- **Does NOT support per-version different messages** in one run. The skill applies one message to the entire range. For per-version messaging, invoke per-version.
- **Does NOT support OIDC-only auth**. Same as `/solo-npm:dist-tag` — local `npm login` required.

## Composition

- `/solo-npm:release` Phase G (post-major release): offers an `AskUserQuestion` gate to deprecate the previous major immediately, with chain-into-this-skill on Yes.
- `/solo-npm:audit` Phase 5 (Tier-1 advisories): expanded options include "Deprecate affected versions" → chain to this skill with the affected version range.
- Manual: invoke directly when you need to retire a single bad version after a botched release (companion to `/solo-npm:dist-tag repoint`).
