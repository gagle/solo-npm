---
description: Unpublish or rename npm packages safely — single version (within 72h or post-72h criteria), full-package removal (--force), or rename-redirect (deprecate old + publish new). Triggers from prompts like "unpublish @wrong/foo@1.0.0", "delete the published v1.0.0", "I shipped under the wrong scope", "rename @gagle/eutils to @ncbijs/eutils", "I changed my mind about the package name", "remove @wrong/foo entirely". Strict safety gates: two AskUserQuestion confirmations for destructive ops; HARD STOP if version has dependents; defaults to deprecate (npm's recommended path).
---

# Unpublish

Recovery skill for the rare-but-real cases where the published artifact itself needs to go away or be redirected — not just deprecated. Use sparingly. **The default-recommended path is `/solo-npm:deprecate` for almost everything**; this skill is the destructive escape hatch with strict safety gates.

## When to use

| Scenario | Right tool |
|---|---|
| Wrong release shipped (bad code, but version number is fine to abandon) | `/solo-npm:dist-tag repoint` + `/solo-npm:deprecate` |
| Vulnerable version that can't be fixed, but tests pass | `/solo-npm:deprecate` (CVE-affected range) |
| Just want to retire an old major | `/solo-npm:deprecate` (range-deprecation) |
| **Wrong package name shipped (typo, wrong scope)** | `/solo-npm:unpublish` (this skill) |
| **Renamed the package (mind change after publish)** | `/solo-npm:unpublish rename-redirect` (this skill) |
| **Accidentally published secrets in tarball** | `/solo-npm:unpublish` (this skill) within 72h **AND credential rotation**. Rotation is non-optional and the load-bearing fix; this skill only handles artifact cleanup. |

## Why npm itself recommends deprecate over unpublish

[npm's official policy](https://docs.npmjs.com/policies/unpublish/) discourages unpublish:

- **Unpublish breaks consumer lockfiles**. Anyone who pinned the unpublished version in `package-lock.json` / `pnpm-lock.yaml` gets a broken `npm install` on next clean install.
- **The same version number cannot be republished** (npm enforces version immutability after the 24h-or-72h window).
- **Deprecation preserves install-ability** — consumers who pinned the bad version keep working but see a `npm WARN` message on every install. They migrate at their own pace.

This skill **defaults to recommending deprecate** in every gate. Unpublish is opt-in destructive.

## Operations

| Operation | Effect |
|---|---|
| `unpublish-version` | Remove a single version (e.g., `@scope/foo@1.0.0`); subject to eligibility |
| `unpublish-all` | Remove the entire package across all versions (`--force`); subject to eligibility |
| `rename-redirect` | Multi-step: deprecate old name with a redirect message + (optional) unpublish old eligible versions. The new package gets published separately via `/init` or `/release` |

## Phase −1 — External-tool reliability (H7 canonical pattern)

Strict-safety baseline applied **before** any other phase. This is the canonical reference for the H7 pattern; sibling skills reference this block. Every external tool the skill will invoke must be verified upfront with three checks: **presence**, **version**, and **timeout-wrappable invocation**. The user gets a clear remediation path if anything fails — no silent hangs, no cryptic later-phase errors.

### −1.1 Tool presence + version detection

```bash
# Required tools for /unpublish:
#   npm   — registry mutations + auth
#   git   — local tag cleanup (Phase D.3 if user opts in)
#   gh    — GitHub Release cleanup (Phase D.3 if user opts in)
#   curl  — deps.dev eligibility check (Phase A.3)
#   jq    — JSON parsing
#   node  — state.json cache cleanup script

require_tool() {  # args: $1 = tool name, $2 = optional minimum-version semver, $3 = canonical install URL
  if ! command -v "$1" >/dev/null 2>&1; then
    echo "ERROR: required tool '$1' is not installed."
    echo "       Install from: $3"
    exit 1
  fi
  # Optional version pin (best-effort — version output formats differ per tool)
  if [ -n "$2" ]; then
    VERSION_OUT=$("$1" --version 2>&1 | head -1)
    echo "  ✓ $1 detected: $VERSION_OUT"
  fi
}

require_tool npm  ""  "https://nodejs.org"
require_tool git  ""  "https://git-scm.com"
require_tool curl ""  "https://curl.se"
require_tool jq   ""  "https://jqlang.github.io/jq/"
require_tool node ""  "https://nodejs.org"

# gh is optional (only used in Phase D.3 if user picks auto-cleanup)
if command -v gh >/dev/null 2>&1 && gh auth status >/dev/null 2>&1; then
  GH_AVAILABLE=1
else
  GH_AVAILABLE=0  # Phase D.3 will skip GitHub Release cleanup with a warning
fi
```

### −1.2 Universal timeout wrapper

Every external network call in this skill **must** be wrappable to bound execution time. Hangs (stale tokens, slow registries, ISP DNS issues) are a strict-safety failure: the user gets no signal that anything is wrong. Convention:

| Call type | Timeout | Why |
|---|---|---|
| `npm whoami` | 30s | Auth check; long enough for slow networks, short enough to surface hung tokens |
| `npm view ...` | 30s | Read-only registry lookups |
| `npm unpublish/deprecate/...` | 60s | Mutations may take longer especially with 2FA flows |
| `curl <api>` | `--max-time 10 --connect-timeout 5` | External APIs: deps.dev, downloads-counter |
| `gh <subcommand>` | 30s | GitHub API |
| `git ls-remote/fetch/push` | 60s | Network git operations |

Implementation: wrap with `timeout(1)` if available (Linux + macOS via `coreutils`), or use the tool's native timeout flag where one exists. Fail-mode: timeout → surface *"<tool> timed out after Ns — network unreachable, token stale, or service slow. Retry, or check manually."*

### −1.3 `.gitconfig` user identity check

```bash
GIT_USER_NAME=$(git config user.name 2>/dev/null)
GIT_USER_EMAIL=$(git config user.email 2>/dev/null)
if [ -z "$GIT_USER_NAME" ] || [ -z "$GIT_USER_EMAIL" ]; then
  echo "ERROR: git config user.name and user.email must be set before running /solo-npm:unpublish."
  echo "       Without them, `git tag -d` operations in Phase D.3 work, but any future commit"
  echo "       (e.g., re-publishing under correct name) will fail with cryptic exit-128 errors."
  echo
  echo "Fix:   git config --global user.name 'Your Name'"
  echo "       git config --global user.email 'you@example.com'"
  exit 1
fi
```

This is checked even though `/unpublish` itself doesn't make commits — it's the entry point check for the H7 pattern, applied universally so the user fixes config issues once. Skills that DO commit (release, prerelease, hotfix, deps, init) absolutely depend on this.

### −1.4 Atomic state.json write helper (used in Phase D.2)

Every state.json write in this skill (and the H7 reference for sibling skills) uses an atomic write pattern: write to a temp file, then rename. Non-atomic `writeFileSync` truncates the file before writing, so a process killed mid-write leaves a zero-byte (or partially-written) cache.

```javascript
// Used wherever the skill writes .solo-npm/state.json:
const tmp = '.solo-npm/state.json.tmp';
fs.writeFileSync(tmp, JSON.stringify(state, null, 2) + '\n');
fs.renameSync(tmp, '.solo-npm/state.json');
// rename() is atomic on POSIX; on the same filesystem, it's a single inode swap.
```

If the process is killed between writeFile and rename, the leftover `.tmp` file is detected on next read (H2 corruption guard already wraps `JSON.parse` in try/catch — the `.tmp` file is harmless because it's not the real state file).

### −1.5 Stale-lock auto-cleanup (extends H5)

The lock file at `.solo-npm/locks/<sanitized>.lock` holds a PID. If the lock-holding process was killed (SIGKILL, OOM, machine crash), the lock persists. Phase C.0's H5 check is upgraded to **auto-clean stale locks** rather than refusing forever:

```bash
LOCK_FILE=".solo-npm/locks/$(echo "<NAME>" | sed 's|/|_|g').lock"
if [ -f "$LOCK_FILE" ]; then
  STALE_PID=$(cat "$LOCK_FILE" 2>/dev/null)
  if [ -n "$STALE_PID" ] && kill -0 "$STALE_PID" 2>/dev/null; then
    echo "ERROR: another /unpublish run holds $LOCK_FILE (PID $STALE_PID is alive)."
    echo "       Wait for it to finish, or kill PID $STALE_PID manually if it's hung."
    exit 1
  fi
  # PID dead or lockfile content unparseable → stale lock; clean it
  echo "WARN: removing stale lockfile $LOCK_FILE (prior holder PID $STALE_PID is no longer alive)"
  rm -f "$LOCK_FILE"
fi
mkdir -p .solo-npm/locks
echo $$ > "$LOCK_FILE"
trap 'rm -f "$LOCK_FILE"' EXIT
# NOTE: trap doesn't fire on SIGKILL — that's why the stale-cleanup above exists.
```

## Phase 0 — Read prompt context

Scan the user's prompt for hints; pre-fill what's clear.

| Prompt mentions | Pre-fill |
|---|---|
| *"unpublish"*, *"delete published"*, *"remove from npm"* | `OPERATION = unpublish-version` (or `-all` if no version named) |
| *"rename X to Y"*, *"@oldscope/foo to @newscope/foo"* | `OPERATION = rename-redirect`, `OLD_NAME = @oldscope/foo`, `NEW_NAME = @newscope/foo` |
| *"wrong scope"*, *"I changed my mind"*, *"shipped under the wrong"* | `OPERATION = unpublish-all` (or rename if both old and new mentioned) |
| `@<scope>/<name>@<version>` | Set `OLD_NAME` + `VERSION` |
| `@<scope>/<name>` (no version) | Set `OLD_NAME` only; default to `unpublish-all` |

Examples:
- *"unpublish @gagel/foo@1.0.0 — typo in the scope"* → `OPERATION=unpublish-version, OLD_NAME=@gagel/foo, VERSION=1.0.0`. Skip prompts; jump to Phase A.
- *"rename @gagle/eutils to @ncbijs/eutils"* → `OPERATION=rename-redirect`. Skip prompts.
- *"I shipped @wrong/foo by mistake — wipe it"* → `OPERATION=unpublish-all, OLD_NAME=@wrong/foo`. Skip prompts.
- Bare *"unpublish"* → AskUserQuestion which package + which version.

## Phase A — Pre-flight + per-version eligibility

### A.1 Auth check

Run `npm whoami`. If exit code != 0 (not authenticated), surface foolproof handoff (matches the pattern in `/trust`, `/dist-tag`, `/deprecate`, `/owner`):

> Run `npm login` to authenticate with npm:
> 1. Type `npm login` and press Enter.
> 2. It prints a URL — open it in your browser.
> 3. Sign in to npmjs.com.
> 4. Approve the login via your authenticator app (web 2FA).
> 5. Return here when the terminal shows your username.

Then `AskUserQuestion`:
```
Header:   "Authenticated"
Question: "Done with `npm login`?"
Options:
  - Yes, continue (Recommended) — re-runs `npm whoami` to verify
  - Abort
```

### A.2 Per-version eligibility check

For each affected version (single version for `unpublish-version`; all published versions for `unpublish-all`; old-name versions for `rename-redirect`):

```bash
# 1. Publish age
PUBLISHED_AT=$(npm view "<NAME>@<VERSION>" time."<VERSION>")
HOURS_SINCE_PUBLISH=$(( (now - publishedAt) / 3600 ))

# 2. Dependents (via deps.dev — free, no auth, integrated in v0.9.0)
#    Capture HTTP status separately to differentiate "not indexed yet" (404)
#    from "API down" (5xx/timeout) — see §A.3 below.
HTTP_STATUS=$(curl -s --max-time 10 --connect-timeout 5 \
  -o /tmp/depsdev.json -w "%{http_code}" \
  "https://api.deps.dev/v3/systems/npm/packages/<NAME>/versions/<VERSION>:dependents")
# --max-time 10: total timeout incl. connect + transfer
# --connect-timeout 5: stop waiting for TCP handshake after 5s
# On timeout, curl exits with code 28 → HTTP_STATUS will be empty → routes into the
# 5xx/timeout STOP branch in the table below (correct strict-safety behavior).
case "$HTTP_STATUS" in
  200) DEPENDENTS=$(jq '.dependentCount // 0' /tmp/depsdev.json) ;;
  404) DEPENDENTS=0; DEPSDEV_NOTE="not-indexed" ;;
  *)   DEPSDEV_NOTE="api-error:$HTTP_STATUS" ;;
esac

# 3. Weekly downloads
DOWNLOADS=$(curl -s --max-time 10 --connect-timeout 5 "https://api.npmjs.org/downloads/point/last-week/<NAME>" | jq '.downloads // 0')
# On timeout/failure: curl exits non-zero, jq receives empty input, returns 0.
# Treating as 0 dl/wk on transient failure is conservative-safe — it makes the version
# MORE eligible for unpublish, not less. The user can manually verify via the npmjs.com
# download stats if the eligibility analysis surprises them.

# 4. Ownership (use --json to avoid header-line off-by-one of `wc -l`)
OWNER_COUNT=$(npm owner ls --json "<NAME>" | jq 'length')
```

Classify each version:

| Classification | Condition | Allowed? |
|---|---|---|
| `eligible-72h` | `HOURS_SINCE_PUBLISH < 72` AND `DEPENDENTS == 0` | ✓ Allowed |
| `eligible-criteria` | `HOURS_SINCE_PUBLISH >= 72` AND `DEPENDENTS == 0` AND `DOWNLOADS < 300` AND `OWNER_COUNT == 1` | ✓ Allowed |
| `blocked-dependents` | `DEPENDENTS > 0` | ❌ HARD STOP — no override |
| `blocked-criteria` | `HOURS_SINCE_PUBLISH >= 72` AND criteria not met | ❌ HARD STOP — no override |

### A.3 deps.dev response handling (4-way)

deps.dev's "no data" can mean two different things — the wrong-name fast-cleanup case is exactly when 404 (not indexed yet) fires, so we must NOT collapse it into "API down":

| HTTP response | Meaning | Treatment |
|---|---|---|
| `200 OK` with `dependentCount` field | Indexed; value is authoritative | Use the value |
| `404 Not Found` for the package | Not indexed yet (typically 5–60min after first publish) | Treat as `dependentCount = 0` AND surface a non-fatal note: *"Note: deps.dev has not yet indexed `<pkg>` (typical 5–60min lag after first publish). Treating as 0 dependents — safe assumption for a package no consumer has had time to install yet."* Continue. |
| `5xx`, `429`, timeout, network error | API genuinely down | **STOP** with: *"Cannot verify dependents — deps.dev API unreachable (HTTP `<status>`). Refusing unpublish without verification (would risk breaking consumers). Try again later, or unpublish manually after verifying via `npm view <pkg> dependents`."* |
| Unexpected schema (no `dependentCount`, malformed JSON) | API contract drift | **STOP** with: *"deps.dev returned unexpected response shape; refusing to proceed without verification. File an issue if this persists."* |

The 404-treatment is safe by definition: a package no consumer has had time to install cannot have dependents. The 5xx/timeout-treatment preserves strict safety: API down → cannot verify → STOP. **The user has no override flag** for the strict-stop cases — bypassing dependent verification would defeat the whole skill. Manual `npm unpublish` is the explicit escape.

### A.4 Render eligibility analysis (always shown)

Before any gate fires. The `Status` column **must name the failing condition with its actual value** when blocked — generic *"blocked (criteria not met)"* hides which knob is the problem and forces the user to dig.

```
Pre-unpublish analysis for @wrong/foo:

  Version    Published         Dependents   Downloads/wk   Owners   Status
  v1.0.0     18h ago            0           12             1        ✓ eligible (within 72h)
  v1.0.1      4h ago            0            0             1        ✓ eligible (within 72h)
  v0.9.5     35d ago            2            8             1        ❌ blocked: dependents (2)
  v0.8.0     90d ago            0          350             1        ❌ blocked: 350 dl/wk (limit 300)
  v0.7.0    120d ago            0           50             2        ❌ blocked: 2 owners (limit 1)

  2 of 5 versions eligible for unpublish.
  3 versions BLOCKED (cannot bypass — would break consumers OR violate npm policy).

  ⚠ npm STRONGLY RECOMMENDS deprecate over unpublish.
     Deprecation preserves install-ability for consumers who pinned this
     version in their lockfile. Unpublish breaks them.
```

Naming rules for the Status column when blocked:

| Failing condition | Status text |
|---|---|
| `DEPENDENTS > 0` (any window) | `blocked: dependents (<N>)` |
| Past 72h AND `DOWNLOADS >= 300` | `blocked: <N> dl/wk (limit 300)` |
| Past 72h AND `OWNER_COUNT > 1` | `blocked: <N> owners (limit 1)` |
| Past 72h AND multiple criteria fail | List all (comma-separated, e.g., `blocked: dependents (2), 350 dl/wk`) |
| deps.dev returned 404 (not indexed yet) | Suffix `Status` value with `(deps.dev not yet indexed)` |

Always show this regardless of operation. Eligibility data drives the gate.

### A.5 NEW_NAME validation (rename-redirect only)

Skip if `OPERATION != rename-redirect`.

For renames, validate that the new name is either available (404) or already owned by the current user. Otherwise the deprecation message *"Renamed to `<NEW_NAME>`"* would point consumers at a name the user can't actually publish to.

```bash
# 1. Check if NEW_NAME exists on the registry
NEW_VIEW_STATUS=$(npm view "<NEW_NAME>" name --json 2>&1; echo "EXIT=$?")
```

| State | Detection | Treatment |
|---|---|---|
| NEW_NAME unregistered (404) | `npm view` exits non-zero with `E404` in stderr | ✓ Fine — user will publish under new name later via `/init` or `/release` |
| NEW_NAME registered AND current user owns it | `npm owner ls --json <NEW_NAME>` parsed; `npm whoami` value present in array | ✓ Fine — user can publish further versions to the new name |
| NEW_NAME registered AND owned by someone else | Owner array does not contain `npm whoami` | ❌ **HARD STOP** — *"`<NEW_NAME>` is registered on npm and is owned by `<other-user>`. The redirect message would point consumers at a package you can't publish to. Pick a different new name, or contact the current owner about a transfer."* |
| `npm view` succeeds but registry response is malformed | jq parse fails | ❌ **HARD STOP** — *"Cannot verify ownership of `<NEW_NAME>` — registry returned unexpected shape. Refusing to proceed."* |

The check is read-only and cheap; runs once per rename invocation.

## Phase B — Two-gate confirmation (destructive ops)

### Gate 1 — choose path (deprecate vs unpublish)

```
Header:   "Recovery path"
Question: "How to proceed with @wrong/foo?"
Options:
  - Deprecate eligible versions (Recommended) — npm's preferred path; consumers who pinned this version keep working but see a WARN message
  - Unpublish eligible versions (DESTRUCTIVE) — removes the artifact; consumers with this version pinned in lockfiles get broken installs
  - Abort
```

If user picks **Deprecate**: chain to `/solo-npm:deprecate` with the eligible version range pre-filled.

**Chain-failure recovery (H6 pattern)**: capture the chained skill's exit signal. If `/solo-npm:deprecate` STOPs internally (empty matching range, message length > 1024, OTP needed and user couldn't provide, etc.), surface the child diagnostic verbatim in `/unpublish` context and present:

```
Header:   "Deprecate chain stopped"
Question: "/solo-npm:deprecate stopped: <verbatim child error>. How to proceed?"
Options:
  - Adjust message/range and retry deprecate
  - Switch to Unpublish path (re-fire Gate 1 with the destructive option)
  - Abort
```

Don't silently swallow the child failure.

If user picks **Unpublish**: proceed to Gate 2.

### Gate 2 — confirm destructive op (only fires after Gate 1 unpublish)

```
Header:   "Confirm unpublish"
Question: "Unpublish @wrong/foo eligible versions? Read carefully:"
Options:
  - I understand and want to proceed — irreversible; same version cannot be republished; consumer lockfiles may break
  - Cancel — let me deprecate instead
```

Two gates make accidental destruction harder. Both must explicitly approve.

### Phase B for `rename-redirect`

For renames, the gate has different shape — orchestration not just destruction:

```
Header:   "Rename plan"
Question: "Rename @gagle/eutils → @ncbijs/eutils?"
Options:
  - Deprecate-only (Recommended) — old name keeps working with a redirect message; consumers can migrate at their pace
  - Deprecate + unpublish eligible versions — clears old where allowed (only for fresh-publish renames within 72h)
  - Abort
```

If `Deprecate + unpublish eligible`: also fires Gate 2 (destructive confirmation) before any unpublish.

The user does NOT publish the new name from this skill — they run `/release` (or `/init` for a fresh repo) under the new name separately. This skill only handles the old name.

### Phase B for `unpublish-all`

For full-package removal, Gate 1 surfaces ALL versions that would be removed:

```
Header:   "Full-package unpublish"
Question: "Remove @wrong/foo entirely (all 3 eligible versions; 1 blocked)?"
Options:
  - Deprecate the entire package (Recommended) — same end-user experience: consumers see deprecation; package disappears from search
  - Unpublish all eligible versions — `npm unpublish @wrong/foo --force`; blocked versions remain on registry
  - Abort
```

Then Gate 2 if user picks unpublish.

If ALL versions are blocked (every one has dependents): the unpublish path is greyed out / unavailable; only deprecate is offered.

## Phase C — Execute

### C.0 Pre-execute: auth-race re-check (H3 pattern) + acquire lock (H5 pattern)

After Gate 2 (the destructive confirmation), the user may have spent minutes deciding. The npm session may have expired during the gate. **Before the first destructive call**, re-verify auth and acquire a per-package lock:

```bash
# 1. Re-check auth (Phase A.1 ran minutes ago; session may be stale)
WHOAMI_NOW=$(npm whoami 2>&1)
if [ $? -ne 0 ] || [ "$WHOAMI_NOW" != "$WHOAMI_AT_PHASE_A" ]; then
  # Surface the same foolproof npm-login handoff from Phase A.1
  # AskUserQuestion: "Done with re-login?"; on Yes loop back to this check
  exit  # Don't proceed to destructive op with stale auth
fi

# 2. Acquire file lock with stale-PID auto-cleanup (per Phase −1.5)
#    Live lock holder → STOP. Dead lock holder → WARN + clean + proceed.
LOCK_FILE=".solo-npm/locks/$(echo "<NAME>" | sed 's|/|_|g').lock"
if [ -f "$LOCK_FILE" ]; then
  STALE_PID=$(cat "$LOCK_FILE" 2>/dev/null)
  if [ -n "$STALE_PID" ] && kill -0 "$STALE_PID" 2>/dev/null; then
    echo "ERROR: another /unpublish run holds $LOCK_FILE (PID $STALE_PID is alive)."
    echo "       Wait for it to finish, or kill PID $STALE_PID manually if it's hung."
    exit 1
  fi
  echo "WARN: removing stale lockfile $LOCK_FILE (prior holder PID $STALE_PID is no longer alive)"
  rm -f "$LOCK_FILE"
fi
mkdir -p .solo-npm/locks
echo $$ > "$LOCK_FILE"
trap 'rm -f "$LOCK_FILE"' EXIT
```

If either check fails, halt before any destructive npm call.

### C.1 Per-version unpublish (for `unpublish-version` and `rename-redirect`)

For each eligible version, with **200ms inter-call backoff**:

```bash
npm unpublish "<NAME>@<VERSION>"
```

Halt on first failure. Surface npm error verbatim. Common failures:

| Error | Meaning | Recovery |
|---|---|---|
| 403 | Auth lapsed during run | Re-authenticate (re-run C.0 auth check); resume |
| 404 | Version doesn't exist (already unpublished?) | Skip; continue to next version |
| 409 | Conflict — version became dependent during the run | Re-classify; this version flips to `blocked-dependents` |
| **EOTP / `OTP required`** in stderr (H1 pattern) | 2FA-on-writes is enabled; npm waits for OTP at stdin; skill cannot type into stdin | Surface manual handoff: *"npm requires an OTP for this operation. Run `npm unpublish <pkg>@<version> --otp=<your-OTP-code>` manually outside the skill, then re-invoke `/solo-npm:unpublish` to clean up state and run remaining versions."* |
| Network error | Transient | Retry once with 1s delay; if persists, halt with clear message |

### C.2 Full-package unpublish (for `unpublish-all`)

```bash
npm unpublish "<NAME>" --force
```

`--force` removes all versions in one call. Same failure handling as C.1, including the EOTP/OTP-required handoff (H1 pattern). If some versions are blocked, the call may partially fail — re-classify after the fact.

### C.3 Rename-redirect deprecation (for `rename-redirect`)

If user picked Deprecate-only OR Deprecate+unpublish, run the deprecation step FIRST so consumers always have a redirect message even if subsequent unpublish fails:

```bash
# Deprecate the entire range with a redirect message:
npm deprecate "<OLD_NAME>" "Renamed to <NEW_NAME>. Install: npm i <NEW_NAME>"
```

The wildcard form (`<OLD_NAME>` without `@<version>`) deprecates ALL existing versions. Future versions of the same name (if accidentally published) won't be auto-deprecated; the user should not publish to the old name again.

**EOTP handling (H1 pattern)**: if the deprecate call hangs awaiting OTP or stderr contains `EOTP`/`OTP required`, surface the same manual-handoff pattern as C.1, adapted: *"Run `npm deprecate <OLD_NAME> '<msg>' --otp=<code>` manually, then re-invoke `/solo-npm:unpublish` to continue."*

If Deprecate+unpublish: after deprecate succeeds, run C.1 per eligible version.

## Phase D — Post-execution

### D.1 Verify (with propagation-lag retry — H4 pattern)

npm CDN can take 30–60s to reflect an unpublish. Don't fail verify on the first 200; **3 attempts × 5s sleep** before treating as inconsistent:

```bash
verify_gone() {  # args: $1 = full coordinate (e.g., "@wrong/foo@1.0.0" or "@wrong/foo")
  for i in 1 2 3; do
    OUT=$(npm view "$1" 2>&1)
    if echo "$OUT" | grep -qE "E404|404 Not Found|^npm ERR! 404"; then
      return 0  # successfully gone
    fi
    [ $i -lt 3 ] && sleep 5
  done
  return 1  # registry still showing data after 3 attempts
}
```

Apply per operation:

- **`unpublish-version`**: call `verify_gone "<NAME>@<VERSION>"` for each unpublished version.
- **`unpublish-all`**: call `verify_gone "<NAME>"` once.
- **`rename-redirect`**: also run `npm view "<OLD_NAME>@<v>" deprecated` for one of the deprecated versions; expect the redirect message.

If `verify_gone` returns 1 after 3 attempts, **don't HARD STOP** — the unpublish itself succeeded; this is verification lag. Surface a non-fatal note: *"Registry not yet reflecting unpublish for `<coord>` after 15s — npm CDN may take up to 5 minutes to propagate. Re-check later with `npm view <coord>`."* Continue to D.2.

### D.2 Cache cleanup (with corruption guard — H2 pattern)

For each unpublished `<NAME>@<VERSION>`, update `.solo-npm/state.json`. Wrap the read in try/catch so a malformed cache file doesn't crash the skill — cache hygiene is best-effort, not load-bearing:

```bash
node -e "
  const fs = require('fs');
  const path = '.solo-npm/state.json';
  let state;
  try {
    state = JSON.parse(fs.readFileSync(path, 'utf8'));
  } catch (e) {
    console.error('WARN: .solo-npm/state.json is malformed or missing; cache cleanup skipped. The unpublish itself succeeded — to regenerate the cache, remove the file and re-run any solo-npm skill.');
    process.exit(0);  // non-fatal
  }

  // Always: remove per-version pkgCheck entry
  if (state.pkgCheck?.lastSize?.['<NAME>@<VERSION>']) {
    delete state.pkgCheck.lastSize['<NAME>@<VERSION>'];
  }

  // unpublish-all only: remove package-level cache entries
  if ('<OPERATION>' === 'unpublish-all') {
    if (state.depsdev?.perPackage?.['<NAME>']) {
      delete state.depsdev.perPackage['<NAME>'];
    }
    if (state.trust?.['<NAME>']) {
      delete state.trust['<NAME>'];  // trust config no longer relevant
    }
    // pkgCheck.lastSize: remove all entries for this package
    if (state.pkgCheck?.lastSize) {
      for (const k of Object.keys(state.pkgCheck.lastSize)) {
        if (k.startsWith('<NAME>@')) delete state.pkgCheck.lastSize[k];
      }
    }
  }

  // Atomic write per Phase −1.4: write to .tmp then rename().
  // Non-atomic writeFileSync truncates first, so a process killed mid-write
  // would corrupt the cache. rename() is a single atomic inode swap on POSIX.
  const tmp = path + '.tmp';
  fs.writeFileSync(tmp, JSON.stringify(state, null, 2) + '\n');
  fs.renameSync(tmp, path);
"
```

**Audit cache is NOT cleared** — advisories are version-keyed AND keyed historical records of what was vulnerable; preserving them is intentional even after unpublish.

### D.3 Optional cleanup gate (git tag + GitHub Release)

After successful unpublish (and before final summary), offer to clean up the git tag and GitHub Release that `/release` Phase C.7 created. These are user-controlled artifacts; **don't auto-delete** — surface a gate.

Build the cleanup target list. For:
- `unpublish-version`: `tags = ["v<VERSION>"]`
- `unpublish-all`: `tags = ["v<v>" for each unpublished version]`
- `rename-redirect`: `tags = ["v<v>" for each version actually unpublished]` (NOT versions that were only deprecated)

If `tags` is empty (nothing to clean up — e.g., rename was deprecate-only), skip the gate and go straight to D.4.

Otherwise fire AskUserQuestion:

```
Header:   "Optional cleanup"
Question: "Also delete the local + remote git tag(s) and GitHub Release(s) for the unpublished version(s)? Tags involved: v1.0.0, v1.0.1"
Options:
  - Yes, both — delete git tags + GitHub Releases (Recommended for wrong-name/typo cleanup)
  - Just the git tags — keep the GitHub Releases as historical record
  - No, keep both — leave for manual cleanup or audit trail
  - Abort cleanup — same as "No"
```

**Execution per choice** (handle each tag/release independently — partial success is acceptable; missing `gh` should not block git-tag deletion):

```bash
# For "Yes, both" or "Just the git tags":
for v in <tag-list>; do
  # Local + remote tag deletion
  if git tag -d "$v" 2>&1; then
    git push origin ":refs/tags/$v" 2>&1 || echo "WARN: local tag $v deleted but remote push failed"
  else
    echo "WARN: git tag -d $v failed (already deleted? not present?); skipping remote"
  fi
done

# For "Yes, both" only — also delete GitHub Releases
if command -v gh >/dev/null 2>&1 && gh auth status >/dev/null 2>&1; then
  for v in <tag-list>; do
    gh release delete "$v" --yes 2>&1 || echo "WARN: gh release delete $v failed"
  done
else
  echo "NOTE: gh not installed or not authenticated; GitHub Release cleanup skipped."
  echo "      Install + authenticate from https://cli.github.com, then run:"
  echo "      gh release delete v1.0.0 --yes  # one per tag"
fi
```

**For "No / Abort"**: emit the manual-cleanup snippet so the user has it if they change their mind:

```
Skipping cleanup. To clean up later:

  git tag -d v1.0.0 && git push origin :refs/tags/v1.0.0
  gh release delete v1.0.0 --yes
  # repeat per version
```

Capture the result of each operation (deleted / kept / failed-with-reason) so D.4 can render it.

### D.4 Final summary + next-step suggestions

Layout the summary to surface (a) what was unpublished, (b) cleanup result, (c) next steps:

```
Unpublish complete.

  Unpublished from registry:
    @wrong/foo@1.0.0  (was 18h ago; 12 dl/wk)
    @wrong/foo@1.0.1  (was 4h ago; 0 dl/wk)
  Skipped (blocked):
    @wrong/foo@0.9.5  (blocked: dependents (2); cannot remove — left in place)

  Cleanup (per your choice "Yes, both"):
    git tag v1.0.0  ✓ deleted (local + remote)
    git tag v1.0.1  ✓ deleted (local + remote)
    GitHub Release v1.0.0  ✓ deleted
    GitHub Release v1.0.1  ⚠ skipped (gh not authenticated; install/login from https://cli.github.com)

Next steps:
  - Republish under the correct name when ready: /solo-npm:init (fresh repo) or /release (existing repo, after updating package.json#name)
  - The blocked version (v0.9.5) remains on npm. To handle it, /solo-npm:deprecate with a migration message.
  - Re-check unpublished versions later if registry verify lagged: npm view @wrong/foo@1.0.0
```

For `rename-redirect`:

```
Rename complete.

  @gagle/eutils:
    Deprecated all versions with redirect message: "Renamed to @ncbijs/eutils. Install: npm i @ncbijs/eutils"
    Unpublished v1.0.0, v1.0.1 (within 72h, no dependents)

  Cleanup (per your choice "Just the git tags"):
    git tag v1.0.0  ✓ deleted (local + remote)
    git tag v1.0.1  ✓ deleted (local + remote)
    GitHub Releases — kept (your choice)

Next steps:
  - Publish @ncbijs/eutils: run /solo-npm:init in the new repo, or update package.json#name and run /release
  - Consumers running `npm i @gagle/eutils` will see the redirect message and can migrate at their pace
```

## Hard stops (no override)

Two conditions cannot be bypassed by any flag, gate option, or chain:

| Condition | Message |
|---|---|
| **Any version has dependents (`DEPENDENTS > 0`)** | *"`<NAME>@<VERSION>` has `<N>` dependents (per deps.dev). Cannot unpublish without breaking them. Deprecate instead, or contact those dependents to migrate first. There is no `--force-with-dependents` override in this skill."* |
| **Post-72h, criteria not met (`DOWNLOADS >= 300` OR `OWNER_COUNT > 1`)** | *"`<NAME>@<VERSION>` is past the 72h window and doesn't meet npm's post-72h criteria (downloads `<N>`/wk, owners `<M>`). Per [npm policy](https://docs.npmjs.com/policies/unpublish/), this requires manual contact at `[email protected]`, OR deprecate via `/solo-npm:deprecate`."* |

These are not "warnings the user can override" — they're refusals. The skill exits cleanly without performing any unpublish. The user can still:
- Use `/solo-npm:deprecate` (which works regardless of dependents)
- Use manual `npm unpublish` outside the skill (the skill doesn't prevent the user; it just refuses to participate in unsafe operations)

## What this skill does NOT do

- **Republish under a new name** — `/release` (existing repo) or `/init` (fresh repo) handles the new name's publish flow.
- **Override npm policy** — there's no flag to bypass the 72h rule, dependent check, or download threshold.
- **Auto-chain from `/release`, `/audit`, or any other skill** — destructive enough that direct invocation only.
- **Cherry-pick versions to keep** — `unpublish-version` operates on the user-named version; `unpublish-all` operates on everything (with `--force`); rename operates on the entire old name. No middle ground.
- **Republish the unpublished version** — npm enforces version-number immutability after the 24/72h window; same number cannot be re-used. Bump to the next number.

## Composition

`/solo-npm:unpublish` is **not auto-chained** from any other skill. It's invoked directly when:

- The user types *"unpublish ..."*, *"delete published ..."*, *"rename ... to ..."*, *"I shipped under the wrong scope"*, etc.
- The user discovers a published-secrets emergency and wants to remove the artifact. Rotation is the load-bearing fix (non-optional); this skill only handles the artifact cleanup half.
- The user changes their mind about a package name post-publish.

Other skills that handle adjacent recovery scenarios:

- `/solo-npm:dist-tag repoint` — repoint `@latest` to a different version (NON-destructive; good for "make this version effectively retired without removing it")
- `/solo-npm:deprecate` — mark versions deprecated with a message (NON-destructive; npm's recommended path for almost everything)

`/solo-npm:unpublish` exists for the rare cases where deprecation isn't enough — wrong name, rename, or publishing-secrets emergency.

## Concurrency / locking

This skill acquires a file lock at `.solo-npm/locks/<sanitized-pkg-name>.lock` in Phase C.0 before any destructive npm call. If another `/unpublish` (or `/deprecate`/`/dist-tag`/`/owner` — they share the same lock prefix in v0.10.0+) is already running on the same package, this run halts cleanly with a diagnostic before doing anything destructive. The lock file holds the PID; if a previous run died without cleanup, the user can `rm .solo-npm/locks/<name>.lock` manually.

Single-user, single-session is the supported mode. If you're orchestrating multiple sessions across machines, serialize them yourself — the lock only protects against same-machine races.
