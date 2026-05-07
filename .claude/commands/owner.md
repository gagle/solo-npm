---
description: Manage npm package maintainers (owners) at portfolio scale — add, remove, or list across packages with one prompt. Triggers from prompts like "add @backup-maintainer to all my packages", "show me who can publish each package", "remove @old-collaborator from @ncbijs/eutils", "audit ownership across portfolio". Bus-factor mitigation + ownership transfer + portfolio audit.
---

# Owner

Manage `npm owner` (maintainers with publish rights) across one package or the entire portfolio.

## When to use

- **Bus-factor mitigation**: add a backup maintainer to every package so you're not the sole gate to publishing.
- **Ownership handoff**: transfer ownership when handing off a project.
- **Portfolio audit**: list who can publish each package across the portfolio.
- **Collaborator removal**: remove an old collaborator's publish access.

## Authentication note

`npm owner` mutations require a local `npm login` session (OIDC doesn't cover ownership operations). The skill's Phase A surfaces a foolproof handoff if auth is missing.

## How it works

Three operations: `ls` (read-only), `add`, `rm`.

## Phase 0 — Read prompt context

| Prompt mentions | Pre-fill |
|---|---|
| *"add"*, *"give publish access to"* | `OPERATION = add` |
| *"remove"*, *"revoke"*, *"drop"* | `OPERATION = rm` |
| *"who can publish"*, *"show me owners"*, *"audit ownership"*, *"list maintainers"* | `OPERATION = ls` |
| `@<username>` (an npm handle) | `USER = <username>` |
| Single package name | `SCOPE = <that-package>` |
| *"all my packages"*, *"across the portfolio"*, *"every package"* | `SCOPE = all` |

Examples:
- *"Add @backup-maintainer to all my packages"* → `OPERATION=add, USER=backup-maintainer, SCOPE=all`. Skip prompts.
- *"Show me who can publish each package"* → `OPERATION=ls, SCOPE=all`. Skip prompts (no gate needed for ls).
- *"Remove @old-collaborator from @ncbijs/eutils"* → `OPERATION=rm, USER=old-collaborator, SCOPE=@ncbijs/eutils`. Skip prompts.
- Bare *"manage owners"* → ask for `OPERATION` first.

If the username isn't extractable, AskUserQuestion: *"Which npm username?"* with text input.

## Phase A — Pre-flight + state read

1. **Auth check**: `npm whoami`. If not authenticated, surface the foolproof handoff (matches `/solo-npm:dist-tag` and `/solo-npm:deprecate`).

2. **Workspace discovery** (if `SCOPE=all`): same as other skills.

3. **Per-package owner read**:

   ```bash
   npm owner ls <pkg>
   ```

   Returns lines like `username <email>`. Parse into a list per package.

4. **Compute proposed mutation set**:

   - **add**: target packages where `USER` is NOT already in the owner list.
   - **rm**: target packages where `USER` IS in the owner list. Reject if removing `USER` would leave the package with zero owners.
   - **ls**: just render; no mutations.

5. **Safety rejections** (STOP before Phase B):

   | Condition | STOP message |
   |---|---|
   | `OPERATION=rm` AND `USER` is the only owner of any target package | *"Cannot remove `@<USER>` from `<pkg>` — they are the sole owner. Add another owner first via `/solo-npm:owner add`."* |
   | `OPERATION=rm` AND `USER` is the *current* `npm whoami` AND would leave themselves without access to a package they need | Surface a warning + AskUserQuestion confirmation: *"You're about to remove yourself from <N> packages. Confirm?"* |
   | `OPERATION=add` AND mutation set is empty (already an owner everywhere) | STOP gracefully: *"`@<USER>` is already an owner of all target packages."* |
   | `OPERATION=rm` AND mutation set is empty | STOP gracefully: *"`@<USER>` is not an owner of any target package."* |

## Phase B — Render plan + AskUserQuestion gate

For `ls`: render the portfolio table directly; **no gate** (read-only).

```
Owners across portfolio:

  @ncbijs/eutils:    gabriel <gabriel@example.com>, backup-maintainer <backup@example.com>
  @ncbijs/blast:     gabriel <gabriel@example.com>
  rfc-bcp47:         gabriel <gabriel@example.com>, backup-maintainer <backup@example.com>
  npm-trust:         gabriel <gabriel@example.com>

Coverage:
  Sole-owner packages (bus-factor risk): @ncbijs/blast, npm-trust
  Multi-owner packages: @ncbijs/eutils, rfc-bcp47

Hint: /solo-npm:owner add @backup-maintainer  (covers all packages in one go)
```

For mutations (`add` / `rm`):

```
Proposed owner mutations across <M> packages:

  add @backup-maintainer to:
    @ncbijs/eutils
    @ncbijs/blast
    rfc-bcp47
    npm-trust
    ...

Already-owner (skipped): @ncbijs/etl
```

Then:

```
Header:   "Apply owner changes"
Question: "Add `@<USER>` to <N> packages?"   (or "Remove `@<USER>` from <N> packages?")
Options:
  - Proceed with <N> mutations (Recommended)
  - Abort
```

## Phase C.0 — Error-handling patterns (H1, H2, H4, H5, H6 from `/unpublish` reference)

Before destructive `npm owner` calls, apply the standard solo-npm error patterns. Canonical wording lives in `/unpublish` Phases C.0–D.2; concrete adaptation per pattern:

- **H1 — OTP / 2FA-on-writes**: detect `EOTP` / `OTP required` in `npm owner` stderr. Surface manual handoff: *"npm requires an OTP. Run `npm owner add <user> <pkg> --otp=<your-OTP>` (or `npm owner rm <user> <pkg> --otp=<your-OTP>`) manually outside the skill, then re-invoke `/solo-npm:owner` to resume the remaining mutations."*
- **H2 — `.solo-npm/state.json` corruption guard**: any `JSON.parse(state.json)` read (e.g., trust cache in Phase A.2) wrap in try/catch. On parse fail surface non-fatal warning, continue with empty defaults.
- **H4 — Registry propagation lag retry**: Phase D verify (`npm owner ls --json <pkg>`) uses 3 attempts × 5s sleep before declaring inconsistency. Don't HARD STOP if still inconsistent — surface non-fatal note: *"Registry not yet reflecting owner change after 15s — npm CDN may take up to 5 minutes; re-check later with `npm owner ls <pkg>`."*
- **H5 — Concurrent invocation lock**: at Phase C start (per package), acquire `.solo-npm/locks/<sanitized-pkg-name>.lock`. Refuse to start if another solo-npm skill holds it.
- **H6 — Chain-target failure recovery**: this skill is currently standalone (no auto-chain in or out), but if a future invocation comes from another skill (e.g., a portfolio bus-factor remediation chain), surface any internal STOP verbatim upward.

## Phase C — Execute

For each mutation, with **200ms inter-call backoff**:

```bash
# add:
npm owner add <USER> <pkg>

# rm:
npm owner rm <USER> <pkg>
```

Halt on first failure. Surface npm error verbatim.

## Phase D — Verify + summarize

Re-read `npm owner ls` for each affected package; confirm the owner list matches expectations.

```
Updated <N> packages:

  @ncbijs/eutils:  +backup-maintainer
  @ncbijs/blast:   +backup-maintainer
  rfc-bcp47:       +backup-maintainer
  ...

Bus-factor across portfolio:
  Sole-owner packages: 0  (was 2)
  Multi-owner packages: 4  (was 2)
```

## Failure modes

| Failure | Where | Recovery |
|---|---|---|
| Not authenticated | Phase A.1 | Foolproof `npm login` handoff |
| Unknown npm username | Phase C | npm rejects with "user does not exist"; surface verbatim, abort run |
| Sole-owner removal attempted | Phase A.5 | STOP — instruct to add another owner first |
| Self-removal (current user) | Phase A.5 | Warning + extra confirmation gate |
| npm rate limit | Phase C | Backoff + retry once; otherwise halt |

## What this skill does NOT do

- **Does NOT manage npm orgs or teams**. `npm org` and `npm team` are for paid-tier multi-account workflows; out of scope for solo-dev.
- **Does NOT manage 2FA or token-level permissions**. Owner-level access only — can-publish, can-modify-package — not granular.
- **Does NOT handle GitHub-level repository collaborators**. That's `gh` territory; this skill is npm-only.
- **Does NOT support OIDC-only auth**. Local `npm login` required.

## Composition

- Standalone — no auto-chains from other skills. Invoke directly when ownership state needs to change.
- Read pairs naturally with `/solo-npm:status` (which doesn't currently surface owner state — could be a v0.7.0+ extension).
