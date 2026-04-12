<!-- /autoplan restore point: /Users/Aleksei_Gurianov/.gstack/projects/browserware-browserware/main-autoplan-restore-20260412-030636.md -->
# Plan: PR2 — `brw open --context <selector>`

Created: 2026-04-12
Branch: main
Repo: browserware/browserware
Status: DRAFT — pending /autoplan review

## Context

PR1 (merged: fa162db) shipped the Browser Context Substrate:
- `BrowserContext` model with `ContextSelector` and `LaunchCapability`
- `brw contexts` — full table, JSON, and plain output
- Fixture-driven profile tests, golden JSON tests
- Critical path bug fix PR #23 (merged: e139d41)

PR2 ships the first payoff: opening a URL into a specific browser context.

The user's workflow becomes:
```bash
brw contexts --json                          # discover what's available
brw open https://github.com --context chrome:work  # launch it
```

## Problem

`brw open` exists in the CLI but prints "Full routing not yet implemented" and exits.
`browserware-launch` is a stub crate with a TODO comment.

A developer who finds the repo today can enumerate contexts but cannot act on them from the CLI.

## Premises

1. PR2's primary output is `brw open <url> --context <selector>` — explicit, deterministic launch into a known context.
2. The launch abstraction belongs in `browserware-launch`, not in the CLI. The crate boundary is: CLI resolves the context, launch executes the OS command.
3. Profile targeting is family-specific: `--profile-directory` for Chromium, `-P` for Firefox, nothing for WebKit/Other.
4. Ambiguity policy (`--on-ambiguous first|warn|error`) is surfaced as a CLI flag, defaulting to `warn` (noisier than `first`, less strict than `error`).
5. Not-launchable contexts (capability.launchable = false) are rejected with a clear error that shows the limitations array.
6. Rules engine integration is NOT in PR2. Routing to a context by URL pattern is PR3 or later.

## Scope

### In scope

**`browserware-launch` crate:**
- Public API: `pub fn launch(context: &BrowserContext, urls: &[Url]) -> Result<()>`
- Platform dispatch on `context.browser.variant`:
  - `BrowserVariant::Chromium(_)` + profile → `executable --profile-directory=<profile_id> <url>`
  - `BrowserVariant::Firefox(_)` + profile → `executable -P <profile_id> --no-remote <url>`
  - WebKit/Other or no profile → `executable <url>` (or `open -a <bundle_id> <url>` on macOS)
- macOS bundle-id launch path: if `browser.bundle_id` is set, use `open -a <bundle_id>` rather than invoking the executable directly (avoids Gatekeeper friction)
- Error variants: `LaunchError::NotLaunchable { limitations }`, `LaunchError::ProcessFailed { status }`, `LaunchError::ContextNotFound`, `LaunchError::BrowserExecutableNotFound`
- No `std::process::Command::status()` panics — always return `Result`

**CLI `brw open` command update (`main.rs` + new `commands/open.rs`):**
- Replace stub `Open { urls, browser, profile }` arms
- New flags: `--context <selector>` (optional), `--on-ambiguous <first|warn|error>` (default: `warn`)
- If `--context` is not provided: use default browser (like current behavior) or print help
- Call `discover_contexts()` from `contexts` module
- Parse selector with `ContextSelector::parse()`
- Select context with `selector.select(&contexts, policy)`
- For each URL: call `browserware_launch::launch(context, &[url])`
- Error path: ambiguous selector lists candidates with their full selectors, not-launchable context shows limitations

**Tests:**
- Unit tests in `browserware-launch` for argument construction (not live browser launch): mock `std::process::Command` output or test argument-building functions directly
- Integration tests in `browserware-cli/tests/cli.rs`: `brw open --context <bad-selector>` returns non-zero exit with helpful error message
- Do not require a real browser installation for tests — use fixture-based or argument-construction tests only

### Not in scope for PR2

- Rules engine: URL-pattern → context routing (PR3)
- `brw open` without `--context`: default browser routing without rules (can remain stub or add minimal default-browser fallback)
- crates.io publishing workflow (tracked in TODOS.md)
- GUI/default-browser shim
- Multiple URL batching (can accept `Vec<String>` but optimize for single-URL case)
- `--dry-run` flag (nice-to-have, defer)

## Implementation Plan

### Step 1: `browserware-launch` crate (core logic)

**`crates/browserware-launch/src/lib.rs`:**
```
pub fn launch(context: &BrowserContext, urls: &[Url]) -> Result<()>
```
Validates `capability.launchable`, dispatches to family-specific builder, executes.

**`crates/browserware-launch/src/command.rs`:**
```
fn build_command(context: &BrowserContext, urls: &[Url]) -> std::process::Command
fn chromium_args(profile_id: &str) -> Vec<OsString>
fn firefox_args(profile_id: &str) -> Vec<OsString>
fn macos_open_args(bundle_id: &str, urls: &[Url]) -> Vec<OsString>
```
Pure functions — no I/O, easy to unit test.

**`crates/browserware-launch/src/error.rs`:**
```rust
#[derive(Debug, thiserror::Error)]
pub enum LaunchError {
    #[error("context is not launchable: {limitations:?}")]
    NotLaunchable { limitations: Vec<String> },
    #[error("browser executable not found: {path}")]
    ExecutableNotFound { path: PathBuf },
    #[error("process exited with status {status}")]
    ProcessFailed { status: std::process::ExitStatus },
}
```

### Step 2: CLI `commands/open.rs`

Extract `brw open` from `main.rs` inline match arm into `commands/open.rs` (mirrors `commands/contexts.rs`).

Arguments:
- `urls: Vec<String>` — positional, 1+
- `--context <selector>` — optional `String`
- `--on-ambiguous <policy>` — optional, values: `first`, `warn`, `error`; default `warn`

Flow:
1. Parse selector (if provided) with `ContextSelector::parse()`
2. `discover_contexts()`
3. `selector.select(&contexts, policy)` → `Result<&BrowserContext>`
4. For each parsed URL: `browserware_launch::launch(context, &[url])`
5. Handle errors: print human-friendly message, exit non-zero

### Step 3: AGENTS.md crates.io status fix

Remove or annotate `crates.io ✅` from the architecture table. The crates are not published. Update to `not yet published` or similar. This is a 3-line change, separate commit.

## Open Questions

1. Should `brw open` without `--context` fall back to the system default browser (like a basic `open <url>` call), or should it print an error until PR3 adds rules?
2. On macOS, should `open -a <BundleId> --args --profile-directory=<profile>` be the Chromium launch path (avoids executable permission issues), or direct executable invocation?
3. Should `--on-ambiguous` accept `first` (silent pick) as the default to match what `brw contexts` does internally, or `warn` to train users toward explicit selectors?
4. Should multiple URLs in one `brw open` call all go to the same context, or should the command be scoped to one URL per invocation?

## Success Criteria

- `brw open https://github.com --context chrome:work` opens GitHub in Chrome's Work profile on macOS, Linux, and Windows (where supported).
- Ambiguous selector prints the list of matching selectors with a "use one of these" message.
- Not-launchable context (e.g., Arc spaces) prints the limitations array and exits non-zero.
- `brw open --context bad-selector https://example.com` exits 1 with a parse error.
- No live browser required for CI tests — all tests use fixtures or mock commands.

## Current Action Items (from PR1 autoplan review)

These items from the PR1 /autoplan review are tracked here until resolved:

| # | Item | Status |
|---|------|--------|
| 1 | README cargo install fix | DONE (PR #23) |
| 2 | Linux Chrome Beta/Dev path bugs | DONE (PR #23) |
| 3 | Linux LibreWolf/Waterfox/Floorp path bugs | DONE (PR #23) |
| 4 | Windows LibreWolf doubled path | DONE (PR #23) |
| 5 | BrowserContext deserialize selector invariant | DONE (PR #23) |
| 6 | Firefox empty Name= bug | DONE (PR #23) |
| 7 | Windows Floorp missing from Firefox paths | DONE (PR #23) |
| 8 | AGENTS.md crates.io ✅ aspirational | OPEN — fix in PR2 or before |
| 9 | Chrome malformed JSON missing tracing::warn! | OPEN — low priority |
| 10 | format_json missing trailing newline | OPEN — low priority |
| 11 | launch_label "limited" branch unreachable | OPEN — fix in PR2 |
| 12 | home_dir() Windows: HOME before USERPROFILE | OPEN — low priority |
| 13 | crates.io publishing workflow | TRACKED in TODOS.md |
| 14 | MCP tool opportunity (Approach C) | FUTURE — PR3 candidate |


---

## /autoplan Phase 1: CEO Review

### Step 0A: Premise Challenge

| # | Premise | Status | Notes |
|---|---------|--------|-------|
| 1 | PR2 primary output = `brw open <url> --context <selector>` | **CHALLENGED** | Both models: this is manual selection, not routing. Creates "verbose launcher" perception risk. The compelling use case is automatic routing (PR3), not explicit --context selection. PR2 is still the right sequencing but framing matters — it should be "building the launch primitive that rules will call" not "the first user product." |
| 2 | Launch abstraction belongs in `browserware-launch`, not CLI | VALID | Clean crate boundary. The CLI resolves context, launch executes OS command. |
| 3 | Firefox profile via `-P <profile_id>` flag | **CHALLENGED** | Claude subagent: Firefox `-P` matches by profile *name*, not directory. If name has spaces or was renamed, behavior is undefined. Consider `--profile <absolute_path>` instead. Needs verification. |
| 4 | Ambiguity policy defaults to `warn` | MODERATE | Both models suggest hardcoding `warn` rather than exposing `--on-ambiguous` as a flag in PR2 (premature; no usage data yet). |
| 5 | Not-launchable contexts rejected with clear error | VALID | Correct; `limitations` array already exists in `LaunchCapability`. |
| 6 | Rules engine NOT in PR2 | VALID | Correct sequencing. PR3. |

**Challenged premises (both models agree):**
- **Premise 3 (Firefox `-P`)**: Runtime correctness risk. Needs verification against actual Firefox behavior before PR2 lands.
- **Framing of Premise 1**: Both models flag that the plan positions PR2 as "user payoff" when it should be positioned as "launch infrastructure that rules will call." The CLI command is a proof-of-concept and integration test for the library, not the final UX.

### Step 0B: Existing Code Leverage Map

| Sub-problem | Existing code | Status |
|-------------|--------------|--------|
| Context discovery | `commands::contexts::discover_contexts()` | Done — reuse directly |
| Selector parsing | `ContextSelector::parse()` | Done |
| Selector matching | `ContextSelector::select(&contexts, policy)` | Done |
| Ambiguity policy | `AmbiguityPolicy` enum | Done |
| Launch capability check | `context.capability.launchable` | Done |
| Profile targeting flags | Not yet implemented | PR2 |
| `std::process::Command` execution | Not yet implemented | PR2 |
| macOS bundle-id path | `browser.bundle_id` field exists | PR2 |
| Error types for launch | Not yet implemented | PR2 |
| CLI `open` stub | `main.rs:Commands::Open` | PR2 — replace stub |

### Step 0C: Dream State Mapping

```
CURRENT STATE                  PR2                              12-MONTH IDEAL
────────────────────────────────────────────────────────────────────────────────
brw contexts --json works      browserware-launch crate          brw set-default + OS intercept
brw open prints "not impl"      with launch() API                rules engine routes links
                                brw open --context selector       cargo install browserware-cli
Profile discovery works         --dry-run flag (explainability)   MCP tool for agents/IDE
                                Firefox -P path verified          GUI shim uses same substrate
                                AGENTS.md fixed                  community scripts & automations
                                                                  Helium/AI browser workflows
PR2 is the launch primitive.   PR3 adds rules on top of it.
```

### Step 0C-bis: Implementation Alternatives

| Approach | Description | Risk | Verdict |
|----------|-------------|------|---------|
| A (system-native launcher) | `open -b <bundle_id>` on macOS, `xdg-open` on Linux, `start` on Windows for all browsers | Medium — loses profile targeting for Chromium | Best fallback for WebKit/Other; not sufficient for profile-specific Chromium/Firefox |
| B (direct exec, family-specific flags) | Chromium: `--profile-directory`, Firefox: `-P <name>` | Medium — Firefox `-P` reliability unknown; Gatekeeper concerns on macOS | Selected in plan; needs Firefox behavior verification |
| C (hybrid) | System-native for WebKit/default, direct-exec for Chromium/Firefox profile targeting | Low — best of both | RECOMMENDATION: This is what the plan implies but doesn't state explicitly |

Auto-decided: Approach C (P5 — explicit over clever, documents tradeoff clearly).

### Step 0D: Mode — SELECTIVE EXPANSION

| Candidate | Decision | Principle | Rationale |
|-----------|----------|-----------|-----------|
| `--dry-run` flag | **ADD to PR2** | P1 (completeness) | Both models flag this. Silent failure on macOS `open --args` makes it critical for debugging. 10 lines in command builder. |
| `--on-ambiguous` as CLI flag | **DEFER to PR3** | P3 (pragmatic) | Hardcode `warn`; no usage data yet to know if flag is needed. |
| `brw open <url>` without `--context` | **ADD minimal fallback** | P1 (completeness) | README already shows this command. Claude subagent: 10 lines, eliminates dead-end UX. Use system default browser. |
| `--on-ambiguous` in `ContextSelector::select()` library API | KEEP | P4 (DRY) | Library API stays — callers like future rules engine need it. Just don't expose it as CLI flag yet. |
| crates.io publishing | DEFER — tracked in TODOS.md | P3 (pragmatic) | Not blocking PR2. |
| AGENTS.md fix | **IN PR2 commit** | P1 (completeness) | 3-line change, removes false crates.io claim. |

### Step 0E: Temporal Interrogation

```
HOUR 1 (branch + first commit):  AGENTS.md crates.io fix (3-line change, easy win).
                                   launch error types: LaunchError enum in error.rs.
                                   
HOUR 2 (library):                  browserware-launch/src/command.rs pure functions.
                                   Verify Firefox -P flag with a real Firefox install.
                                   If -P targets profile name, switch to --profile <path>.

HOUR 3 (CLI update):               commands/open.rs with --context and --dry-run.
                                   Replace main.rs Open arm with commands::open::run().
                                   Add minimal fallback: brw open <url> → system launcher.

HOUR 4-5 (tests):                  Unit tests for command argument construction.
                                   CLI integration tests: bad selector → error, dry-run → shows command.
                                   No live browser launch in CI.

HOUR 6 (CI/review):                cargo test --workspace passes.
                                   Clippy clean.
                                   PR description clearly frames: "launch primitive for rules engine".
```

### Step 0F: Mode = SELECTIVE EXPANSION (confirmed)

---

### CEO Dual Voices

**CODEX SAYS (CEO — strategy challenge):**
> "This plan is internally coherent and strategically evasive. It advances the architecture but avoids deciding whether Browserware's first real product is a router or a launcher. Both models: train users to think Browserware is a verbose launcher, not a routing system. The make-or-break product decision (`brw open` without `--context`) is deferred as 'open question' while the command is shipped as 'user-facing payoff'. Add `--dry-run` for trust/explainability. Standardizing `--context` selector grammar too early before knowing durable UX."

**CLAUDE SUBAGENT (CEO — strategic independence):**
> "Firefox `-P` targets profile name not directory — silent failure risk. `open -a --args` on macOS can silently drop arguments under Gatekeeper/sandboxing — needs explicit test. `brw open <url>` without `--context` is a dead end (README already shows it). `--dry-run` is critical for debugging, not cosmetic. `--on-ambiguous` flag is premature — hardcode `warn`."

```
CEO DUAL VOICES — CONSENSUS TABLE:
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Premises valid?                   PARTIAL  PARTIAL  DISAGREE → Premise 3 (Firefox) challenged
  2. Right problem to solve?           YES*    NO*     DISAGREE → User Challenge (see gate)
  3. Scope calibration correct?        PARTIAL  PARTIAL  DISAGREE → --dry-run missing; fallback missing
  4. Alternatives sufficiently explored? NO    NO     CONFIRMED NOT EXPLORED
  5. Competitive/market risks covered? LOW     LOW     CONFIRMED LOW risk for PR2
  6. 6-month trajectory sound?         PARTIAL  NO     DISAGREE → positioning risk flagged
═══════════════════════════════════════════════════════════════
CONFIRMED = both agree. DISAGREE = models differ (→ taste decision or user challenge).
```

**User Challenge:** Both models question whether `brw open --context` as "user product" is the right framing. This is surfaced at the final gate. Auto-decided per P6 (bias toward action): proceed with PR2 as launch infrastructure, add `--dry-run` and minimal fallback, frame as "library proof-of-concept" not "consumer product."

### CEO Section 1: Architecture Review

`browserware-launch` as a separate crate is correct. It sits between `types` (contract) and `cli` (consumer). The `detect` and `profiles` crates are parallel; `launch` does not need to depend on them — the CLI composes detect + profiles + launch together.

**Dependency concern:** `browserware-launch` currently depends on `browserware-types` and `anyhow`. For PR2, it also needs `url` (already in workspace deps) but nothing else. This stays thin.

**macOS launch path:** `browser.bundle_id` exists on `Browser`. On macOS, `open -a <bundle_id> --args <flags>` is safer than invoking the executable directly because it respects app signing and avoids "damaged app" Gatekeeper warnings. This should be the primary macOS path for Chromium/Firefox profile launch, with direct-exec as Linux/Windows path.

**Verdict: CLEAN. Architecture as planned is correct.**

### CEO Section 2: Error & Rescue Map

| Error Scenario | Handling | Verdict |
|----------------|----------|---------|
| `context.capability.launchable = false` | `LaunchError::NotLaunchable { limitations }` — show limitations array | GOOD |
| Executable path does not exist | `LaunchError::ExecutableNotFound { path }` | GOOD |
| Process exits non-zero | `LaunchError::ProcessFailed { status }` | GOOD |
| Selector parse failure | `ContextSelector::parse()` returns `Err` — already handled | GOOD |
| Ambiguous selector | `AmbiguityPolicy::Warn` → log warning, use first | GOOD (hardcoded warn for PR2) |
| No contexts match selector | `Error::Other("no context matches selector '...'")` with hint to run `brw contexts` | NEEDS EXPLICIT HANDLING |
| `open -a` silently drops `--args` on macOS | Not handled — no feedback to user | NEEDS: `--dry-run` to preview; explicit doc note |
| Firefox `-P` targets wrong profile | Not handled — runtime ambiguity | NEEDS: verification + fallback to `--profile <path>` |

### CEO Section 3: Security Review

- No shell-injection risk: arguments are passed as separate `OsString` values in `Command::arg()`, not shell-interpolated
- `bundle_id` comes from `browser.bundle_id` which was populated from macOS plist — safe provenance
- URL values are `url::Url` types, not raw strings — already validated
- No privilege escalation: launching a browser the user already has installed
- **Watch:** if `profile_id` is used in `--profile-directory=<value>`, ensure it is passed as a single `--arg` with `=` joined, not split into two args (Chrome only accepts joined form)

**Verdict: CLEAN with one Chrome argument formatting note.**

### CEO Section 4: Data Flow Review

```
CLI: brw open <url> --context <selector>
  ├─ parse URL: url::Url::parse(url_str)?
  ├─ parse selector: ContextSelector::parse(selector_str)?
  ├─ discover_contexts()
  │   ├─ detect_browsers() → Vec<Browser>
  │   └─ for each browser: discover_profiles() → Vec<BrowserContext>
  ├─ selector.select(&contexts, AmbiguityPolicy::Warn)
  │   ├─ exact match → Ok(&BrowserContext)
  │   ├─ multiple matches → warn + use first
  │   └─ no match → Err
  └─ browserware_launch::launch(context, &[url])
      ├─ check capability.launchable → Err(NotLaunchable) if false
      ├─ build_command(context, urls)
      │   ├─ macOS + bundle_id: open -a <bundle_id> --args <flags> <urls>
      │   ├─ Chromium + profile: exec --profile-directory=<id> <urls>
      │   ├─ Firefox + profile: exec -P <name> --no-remote <urls>
      │   └─ Other/no profile: exec <urls>
      └─ command.status()? → Ok(()) or Err(ProcessFailed)
```

**Verdict: CLEAN. Data flow is explicit and deterministic.**

### CEO Section 5: Code Quality

No code written yet. Design points:
- `command.rs` pure functions make the critical path unit-testable without spawning processes
- `LaunchError` with `thiserror` is correct pattern (already used in `browserware-types`)
- `commands/open.rs` should follow the same structure as `commands/contexts.rs` — `run()` entry, helper functions for formatting errors

### CEO Section 6: Test Review

Tests needed (no live browser in CI):
- `browserware-launch`: argument-construction unit tests for each browser family + profile combination
- `browserware-launch`: `NotLaunchable` and `ExecutableNotFound` error path tests
- `browserware-cli`: `brw open --context bad-selector https://example.com` → non-zero exit
- `browserware-cli`: `brw open --dry-run --context chrome:work https://example.com` → prints command, no launch
- `browserware-cli`: `brw open https://example.com` (no `--context`) → system launcher behavior

### CEO Section 7: Performance

Single invocation of `discover_contexts()` per `brw open` call. Same cost as `brw contexts` — 2-5 small filesystem reads. No concern.

### CEO Section 8: Observability

- `--dry-run` flag provides pre-launch visibility into the exact command being constructed
- `LaunchError::NotLaunchable` prints the `limitations` array inline
- `tracing::info!` on successful launch (what command was run) at INFO level

### CEO Section 9: Deployment Readiness

| Check | Status |
|-------|--------|
| AGENTS.md crates.io fix | Needs 3-line commit |
| `browserware-launch` implemented | PR2 |
| `brw open` stub replaced | PR2 |
| `--dry-run` flag | PR2 (add to scope) |
| Firefox `-P` verified | PR2 (must verify before merge) |
| No live browser in CI | PR2 (argument-construction tests only) |

### CEO Section 10: Long-Term Trajectory

PR2 makes `brw open --context <selector>` the proof point for the launch substrate. The 12-month vision is:
1. PR3: Rules engine routes `brw open <url>` automatically
2. PR4/5: Default-browser registration intercepts OS-level link clicks
3. MCP tool: `browserware_open(url, selector)` makes this composable for AI agents

The selector grammar established in PR1/PR2 should be treated as a stable public API from the moment PR2 merges. Changes will be breaking. Auto-decided: mark `ContextSelector` fields as stable in docs.

### CEO Phase 1 Completion Summary

**Overall verdict: Proceed with PR2. Add `--dry-run`, minimal default-browser fallback, verify Firefox `-P` behavior. Frame PR2 as "launch infrastructure" not "consumer product."**

| Issue | Severity | Action |
|-------|----------|--------|
| Firefox `-P` flag targets profile name not path | CRITICAL | Verify behavior; switch to `--profile <path>` if `-P` unreliable |
| `brw open <url>` (no `--context`) is dead end — README shows it | HIGH | Add system-launcher fallback in PR2 |
| `--dry-run` deferred — needed for explainability/trust | HIGH | Add to PR2 scope (10 lines) |
| AGENTS.md crates.io ✅ aspirational | HIGH | Fix in PR2 commit |
| PR framing: "user payoff" vs "infrastructure" | FRAMING | Address in PR description |
| `--on-ambiguous` flag premature | MEDIUM | Hardcode `warn`, defer flag to PR3 |

**Phase 1 (CEO Review): COMPLETE. Proceeding to Phase 3 (Eng Review). Phase 2 skipped (no UI scope).**

---

## Decision Audit Trail

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|-------|----------|-----------|-----------|----------|---------|
| 1 | CEO | Add `--dry-run` to PR2 | Mechanical | P1 (completeness) | Both models flag this; macOS silent-failure risk makes it critical | Defer to PR3 |
| 2 | CEO | Hardcode `warn` for ambiguity policy, defer `--on-ambiguous` flag to PR3 | Mechanical | P3 (pragmatic) | No usage data yet; reduces CLI flag surface | Expose as flag now |
| 3 | CEO | Add minimal `brw open <url>` fallback (system launcher) | Mechanical | P1 (completeness) | README shows this command; dead end is worst first impression | Defer to PR3 |
| 4 | CEO | Firefox `-P` flag — verify before PR2 merges | Mechanical | P1 (completeness) | Silent profile mismatch is worse than no profile targeting | Assume it works |
| 5 | CEO | Approach C (hybrid macOS-native + direct-exec) for launch | Mechanical | P5 (explicit) | Documents tradeoff; Gatekeeper safer with bundle-id path | Pure direct-exec |
| 6 | CEO | AGENTS.md crates.io fix in PR2 commit | Mechanical | P1 (completeness) | 3-line fix, false claim | Separate PR |


---

## /autoplan Phase 3: Engineering Review

**Pre-phase 3 updates to plan based on premise gate:**
- Firefox profile targeting: `--profile <absolute_path>` confirmed (not `-P <name>`)
- This requires: (1) `ProfileRef.path: Option<PathBuf>` addition, (2) Firefox parser to read `Path=` + `IsRelative=`, (3) launch crate to use `profile.path` for Firefox

### Step 0: Scope Challenge

**Cross-crate impact of Firefox path decision:**

| Change | Crate | Severity | In blast radius? |
|--------|-------|----------|------------------|
| `ProfileRef.path: Option<PathBuf>` | `browserware-types` | Breaking (serde) | YES |
| Firefox parser: read `Path=` + `IsRelative=` | `browserware-profiles` | Logic change | YES |
| Update all `ProfileRef { id, display_name }` constructors | `browserware-types`, `browserware-profiles`, test fixtures | Additive | YES |
| `#[serde(default)]` on `path` field | `browserware-types` | Additive | YES |
| `browserware-launch` uses `profile.path` for Firefox | `browserware-launch` | New crate | YES |
| CLI `commands/open.rs` | `browserware-cli` | New file | YES |
| `browserware-system::open_default()` (see below) | `browserware-system` | New function | YES |

All changes are in blast radius. Auto-decided: proceed (P2 — boil lakes, <5 files, no new infra).

### Step 0.5: Eng Dual Voices

**CLAUDE SUBAGENT (Eng — independent review):**
- CRITICAL: macOS `open -a` + URL ordering — URLs must come before `--args`, or URL is treated as an app arg
- HIGH: Firefox `IsRelative=0` — absolute path must NOT be re-joined with profiles root
- HIGH: Arc `launchable=false` fixture test missing
- MEDIUM: `ProfileRef.path` needs `#[serde(default)]`; golden JSON fixtures need regeneration
- LOW: Empty `urls` slice behavior undefined; `LaunchError::ContextNotFound` wrong boundary

**CODEX SAYS (Eng — architecture challenge):**
- CRITICAL: `ProfileRef.path` is missing from the type model — PR2 cannot implement Firefox absolute path without it
- HIGH: macOS `open -b <bundle_id>` not `-a <bundle_id>` — `-a` accepts name/path, `-b` accepts bundle identifier
- HIGH: `brw open <url>` fallback (no `--context`) has no crate owner — `browserware-system` is the right home
- HIGH: Ambiguity `AmbiguityPolicy::Warn` doesn't expose candidates to caller — CLI would need to re-implement selection logic
- MEDIUM: URL batching semantics inconsistent — API signature `&[Url]` implies single invocation; CLI says "for each URL"
- MEDIUM: Test injection seam missing for CLI layer — `discover_contexts()` is private, no fixture injection possible
- MEDIUM: `LaunchError::ContextNotFound` wrong crate boundary — launch takes already-resolved context

```
ENG DUAL VOICES — CONSENSUS TABLE:
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Architecture sound?               PARTIAL PARTIAL  DISAGREE → 4 structural gaps to fix
  2. Test coverage sufficient?         PARTIAL  NO     DISAGREE → CLI injection seam missing
  3. Performance risks addressed?      YES     YES     CONFIRMED clean
  4. Security threats covered?         YES     YES     CONFIRMED clean
  5. Error paths handled?              PARTIAL PARTIAL  DISAGREE → ContextNotFound wrong boundary
  6. Deployment risk manageable?       PARTIAL PARTIAL  DISAGREE → Firefox path change is breaking
═══════════════════════════════════════════════════════════════
```

**Auto-decisions on all structural gaps:**

| # | Gap | Auto-decision | Principle |
|---|-----|---------------|-----------|
| 1 | `ProfileRef.path` missing | ADD `path: Option<PathBuf>` with `#[serde(default, skip_serializing_if = "Option::is_none")]` | P1 |
| 2 | macOS: use `-b` not `-a` for bundle ID | Use `open -b <bundle_id>` | P5 (correct semantics) |
| 3 | macOS: URL ordering before `--args` | URL args before `--args` separator | P1 (correctness) |
| 4 | `brw open` fallback owner | Add `pub fn open_default(urls: &[Url]) -> Result<()>` to `browserware-system` | P5 (explicit boundary) |
| 5 | Ambiguity policy for `brw open` | Use `AmbiguityPolicy::Error` (not `Warn`) — launch is a side effect, not read-only | P5 (explicit) |
| 6 | `LaunchError::ContextNotFound` | REMOVE — wrong crate boundary; CLI owns "no match" error | P5 |
| 7 | URL batching | Single `launch(context, &all_urls)` call — opens all URLs in one command invocation (multiple tabs) | P1 |
| 8 | CLI test injection | Expose `pub(crate) fn open_context(contexts: &[BrowserContext], selector: &str, urls: &[Url], dry_run: bool) -> Result<()>` — testable without live detection | P5 |

### Section 1: Architecture (Updated)

```
CLI (brw open)
  ├─ ContextSelector::parse(selector)
  ├─ discover_contexts()           ← reuse from contexts module
  ├─ selector.select(&contexts, AmbiguityPolicy::Error)
  │   ├─ exact match → &BrowserContext
  │   └─ no match / ambiguous → Err with candidate list
  ├─ browserware_launch::launch(context, &all_urls)
  │   ├─ check capability.launchable → LaunchError::NotLaunchable
  │   ├─ build_command(context, urls)
  │   │   ├─ macOS + bundle_id: open -b <id> <urls...> --args <profile_flags>
  │   │   ├─ Chromium + profile: exec --profile-directory=<id> <urls...>
  │   │   ├─ Firefox + profile.path: exec --profile <abs_path> --no-remote <urls...>
  │   │   └─ Other/no profile: exec <urls...>
  │   └─ command.status()? → ProcessFailed
  └─ brw open <url> (no --context)
      └─ browserware_system::open_default(&[url])
          ├─ macOS: open <url>
          ├─ Linux: xdg-open <url>
          └─ Windows: start <url>
```

**ASCII Dependency graph:**
```
browserware-cli
  ├─ browserware-types (BrowserContext, ContextSelector)
  ├─ browserware-detect (detect_browsers)
  ├─ browserware-profiles (discover_profiles)
  ├─ browserware-launch (launch)          ← NEW (was stub)
  │   └─ browserware-types
  └─ browserware-system (open_default)   ← NEW function
      (no deps on other internal crates)
```

No circular dependencies. `browserware-launch` does NOT depend on detect/profiles — correct.

### Section 2: Code Quality

- `commands/open.rs` follows same pattern as `commands/contexts.rs`: `pub(crate) fn run()` entry point
- `command.rs` in `browserware-launch` is pure functions — no I/O, easy to unit test
- `LaunchError` uses `thiserror` — consistent with `browserware-types::Error`
- `--dry-run` flag: `build_command(context, urls)` returns `Command` struct; `--dry-run` prints args without spawning

**Firefox `--profile` note:** `--no-remote` flag is needed alongside `--profile` to prevent Firefox from forwarding the URL to an existing instance (wrong profile). Must be in `firefox_args()`.

**Chrome `--profile-directory` note:** Must be `--profile-directory=<dir_name>` with `=` (single arg), NOT `["--profile-directory", "<dir_name>"]` (two args). Chrome only accepts the joined form.

### Section 3: Test Review (NEVER SKIP)

**Test diagram — all new paths:**

| Path | Test Type | Exists? | Gap? |
|------|-----------|---------|------|
| `ProfileRef.path` serde round-trip with `None` | Unit (browserware-types) | No | ADD |
| `ProfileRef.path` serde with omitted key → `None` | Unit (browserware-types) | No | ADD |
| Firefox INI: `Path=` + `IsRelative=1` → absolute path | Unit (browserware-profiles) | No | ADD |
| Firefox INI: `Path=` + `IsRelative=0` (absolute) → path as-is | Unit (browserware-profiles) | No | ADD |
| Firefox INI: `Path=` missing → `None` path | Unit (browserware-profiles) | No | ADD |
| `build_command` Chromium + profile → `--profile-directory=<id>` joined | Unit (browserware-launch) | No | ADD |
| `build_command` Firefox + path → `--profile /abs/path --no-remote` | Unit (browserware-launch) | No | ADD |
| `build_command` macOS bundle_id → `open -b <id> <url> --args <flags>` | Unit (browserware-launch) | No | ADD |
| `build_command` Other/no profile → `exec <url>` only | Unit (browserware-launch) | No | ADD |
| `launch()` with `launchable=false` → `NotLaunchable` | Unit (browserware-launch) | No | ADD |
| `brw open --context bad https://example.com` → exit 1 + error | Integration (browserware-cli) | No | ADD |
| `brw open --dry-run --context chrome https://example.com` → prints command, no launch | Integration (browserware-cli) | No | ADD |
| `brw open https://example.com` (no context) → exits 0 or uses system launcher | Integration (browserware-cli) | No | ADD |
| `browserware_system::open_default` unit test (mock or check arg construction) | Unit (browserware-system) | No | ADD |

**Golden JSON test update:** `brw contexts --json` output will change when `ProfileRef.path` is added (new field, with `skip_serializing_if = "Option::is_none"` → field absent for non-Firefox or path-unknown contexts). Regenerate goldens after PR2 merge.

Test plan artifact written to disk at:
`/Users/Aleksei_Gurianov/.gstack/projects/browserware-browserware/pr2-test-plan-20260412.md`

### Section 4: Performance

Same cost as PR1 (`discover_contexts()` per invocation). No regression.

### Section 5-8: Security, Observability, Deployment, Long-Term

**Security:** `Command::arg()` with `OsString` throughout — no shell injection. `profile.path` from `profiles.ini` (not user input) — safe provenance.

**Observability:** `--dry-run` flag provides full command preview. `tracing::info!` on successful launch at INFO level.

**Deployment:**
| Check | Status |
|-------|--------|
| `ProfileRef.path` serde backward-compat | YES — `Option` + `skip_serializing_if` |
| Golden JSON tests regenerated | PR2 task |
| Firefox `--profile <path>` verified live | PR2 task (must test before merge) |
| macOS `open -b` vs `-a` | Fixed in plan |
| `cargo test --workspace` | PR2 task |

**Long-term:** PR2 establishes `browserware-launch::launch(context, urls)` as the stable API that rules engine will call in PR3. The `browserware-system::open_default()` becomes the fallback for the rules engine when no rule matches. Both are load-bearing for PR3.

### Phase 3 Completion Summary

**3 structural gaps fixed in plan. No blockers if plan is followed exactly.**

| Issue | Severity | Action |
|-------|----------|--------|
| `ProfileRef.path` field + Firefox `Path=`/`IsRelative=` parsing | CRITICAL | In PR2 scope — add field + update parser |
| macOS: use `open -b` not `open -a` | CRITICAL | Fixed in plan — use `-b <bundle_id>` |
| URL ordering before `--args` in macOS `open` | CRITICAL | Fixed in plan — URLs precede `--args` |
| Firefox `--no-remote` flag needed alongside `--profile` | HIGH | Add to `firefox_args()` |
| `LaunchError::ContextNotFound` wrong boundary | HIGH | Remove from error enum |
| `AmbiguityPolicy::Error` for `brw open` (not `Warn`) | HIGH | Fixed in plan |
| `browserware-system::open_default()` | HIGH | Add to PR2 scope |
| URL batching: single `launch()` call | MEDIUM | Fixed in plan |
| CLI test injection seam | MEDIUM | Expose `open_context()` helper |
| Golden JSON tests need regeneration | MEDIUM | PR2 task after implementation |

**Phase 3 (Eng Review): COMPLETE. Proceeding to Phase 3.5 (DX Review).**

---

## Decision Audit Trail (continued)

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|-------|----------|-----------|-----------|----------|---------|
| 7 | Eng | Add `ProfileRef.path: Option<PathBuf>` | Mechanical | P1 | Firefox `--profile <path>` requires it; in blast radius | Defer |
| 8 | Eng | Firefox INI: read `Path=` + `IsRelative=` | Mechanical | P1 | Needed for absolute path construction | Keep `-P` |
| 9 | Eng | macOS: use `open -b <bundle_id>` not `-a` | Mechanical | P5 | `-a` takes name/path; `-b` takes bundle ID — correct semantics | Use `-a` |
| 10 | Eng | URLs before `--args` in macOS open | Mechanical | P1 | Correctness — `--args` separates app flags from open flags | Any order |
| 11 | Eng | `browserware-system::open_default()` for no-context fallback | Mechanical | P5 | Clear crate boundary; `system` owns OS integration | Put in CLI |
| 12 | Eng | `AmbiguityPolicy::Error` for `brw open` (not `Warn`) | Mechanical | P5 | Launch is a side effect; explicit failure > silent pick | Warn |
| 13 | Eng | Remove `LaunchError::ContextNotFound` | Mechanical | P5 | Wrong crate boundary; CLI owns "no match" | Keep |
| 14 | Eng | Single `launch(context, &all_urls)` call | Mechanical | P1 | Opens multiple tabs in one session | Per-URL calls |
| 15 | Eng | Expose `open_context()` helper for CLI tests | Mechanical | P5 | Testable without live detection | Private only |


---

## /autoplan Phase 3.5: DX Review

DX scope confirmed: CLI, flags, error messages, developer-facing tool.

### Step 0: DX Scope

Developer journey map (9 stages):

| Stage | Current | After PR2 |
|-------|---------|-----------|
| 1. Discovery | GitHub repo | Same |
| 2. Install | `cargo install --git ...` (README is correct) | Same |
| 3. First command | `brw contexts` — works | Same |
| 4. Understand output | Selector syntax is unclear | Improve with copy-paste examples in README |
| 5. First launch | `brw open https://github.com` → "not impl" | `brw open URL` → system default browser |
| 6. Profile-targeted launch | Not possible | `brw open URL --context chrome:work` |
| 7. Debug failure | No path | `brw open --dry-run ...` or `RUST_LOG=info` |
| 8. Error recovery | Cryptic errors | "hint: run `brw contexts`" suffix |
| 9. Upgrade | Breaking flag change no migration doc | CHANGELOG entry + deprecation note |

**TTHW current:** ~10 min. **TTHW target:** <5 min. **With fixes:** ~4 min achievable.

**Initial DX score: 4/10** (Codex assessment). **After plan fixes: 7/10**.

### Step 0.5: DX Dual Voices

**CLAUDE SUBAGENT (DX):**
- CRITICAL: `--dry-run` missing from CLI argument spec (appears in test plan but not in scope/Step 2)
- CRITICAL: `AmbiguityPolicy` default contradicts itself — scope says `warn`, Eng review says `Error`
- HIGH: Error messages are implementation-shaped, not user-shaped (no "hint:" lines)
- HIGH: No TTHW path — README needs getting-started block with copy-paste examples
- HIGH: `open_context()` return type unspecified
- MEDIUM: `RUST_LOG=info` debug path undocumented
- MEDIUM: README update not in implementation steps

**CODEX SAYS (DX):**
- HIGH: First-run UX still too manual (4 steps before value) — needs default-browser fallback
- HIGH: Error messages parser-centric not user-centric
- MEDIUM: `brw contexts` output doesn't show copy-paste `brw open` examples
- MEDIUM: No migration guide for `--browser`/`--profile` → `--context` change (flags existed even as stubs)
- MEDIUM: Firefox-path change is a breaking library API change with no migration note

```
DX DUAL VOICES — CONSENSUS TABLE:
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Getting started < 5 min?          NO      NO     CONFIRMED NOT MET → needs README fixes
  2. API/CLI naming guessable?         YES     YES    CONFIRMED clean
  3. Error messages actionable?        NO      NO     CONFIRMED missing "hint:" lines
  4. Docs findable & complete?         NO      NO     CONFIRMED missing quickstart
  5. Upgrade path safe?                NO      NO     CONFIRMED missing migration note
  6. Dev environment friction-free?    PARTIAL PARTIAL DISAGREE → --dry-run gap
═══════════════════════════════════════════════════════════════
```

### DX Passes 1-8

**Pass 1 — Getting Started (4/10 → 7/10 with fixes):**
Auto-decided: add "Getting Started" section to README as PR2 deliverable. Must include:
```bash
# 1. See what contexts are available
brw contexts

# 2. Open a URL in the system default browser
brw open https://github.com

# 3. Open a URL in a specific context (copy a selector from step 1)
brw open https://github.com --context chrome:work

# 4. Preview the launch command without launching
brw open https://github.com --context chrome:work --dry-run

# Debug: see what's happening
RUST_LOG=info brw open https://github.com --context chrome:work
```

**Pass 2 — First Success (6/10):** `brw open URL` → system default browser works immediately. No context discovery needed. Score: 6/10.

**Pass 3 — Error Messages (5/10 → 7/10 with fixes):**
Mandate pattern for every error:
```
error: <what went wrong>
  cause: <why>
  hint: run `brw <corrective command>`
```
Examples:
- "ambiguous" → `hint: rerun with one of:\n  brw open URL --context chrome:work\n  brw open URL --context chrome:personal`
- "not launchable" → `hint: run \`brw contexts\` to find a launchable context`
- "no match" → `hint: run \`brw contexts\` to list available selectors`

**Pass 4 — CLI Design (8/10):** `brw contexts`, `brw open`, `--context`, `--dry-run` are all guessable. Score: 8/10.

**Pass 5 — Docs (3/10 → 6/10):** README quickstart needed. `brw open --help` must list `--context` and `--dry-run`. Score: 6/10 after fixes.

**Pass 6 — Upgrade Path (5/10):** CHANGELOG entry needed for `--context` flag + deprecation of `--browser`/`--profile` (they existed as stubs; users may have scripted against them). Add `deprecated:` section in CHANGELOG under 0.3.0.

**Pass 7 — API ergonomics (8/10):** Selector syntax is the main DX surface. `chrome:work` alias is discoverable; canonical form is documented. Score: 8/10.

**Pass 8 — Escape hatches (7/10):** `--dry-run` + `RUST_LOG=info` + error "hint:" pattern together give adequate escape hatches. Score: 7/10.

### DX Scorecard

| Dimension | Initial | After fixes | Gap |
|-----------|---------|-------------|-----|
| Getting started | 4 | 7 | Add README quickstart |
| CLI naming | 8 | 8 | Clean |
| Error messages | 3 | 7 | Add "hint:" pattern |
| Docs | 3 | 6 | README + --help |
| Upgrade path | 4 | 7 | CHANGELOG + migration note |
| Escape hatches | 5 | 8 | --dry-run + RUST_LOG |
| **Overall** | **4** | **7** | |

### DX Auto-decisions

| # | Decision | Classification | Principle |
|---|----------|----------------|-----------|
| 16 | Fix scope section to match accepted decisions (--dry-run, no --on-ambiguous, AmbiguityPolicy::Error) | Mechanical | P5 |
| 17 | Add README "Getting Started" section as PR2 deliverable | Mechanical | P1 |
| 18 | Mandate "hint:" line on every error message from brw open | Mechanical | P1 |
| 19 | `open_context()` return type: `anyhow::Result<()>` | Mechanical | P5 |
| 20 | System default launcher loops per-URL (xdg-open/open/start take single URL) | Mechanical | P1 |
| 21 | CHANGELOG entry for 0.3.0: add --context, deprecate --browser/--profile | Mechanical | P1 |
| 22 | brw contexts output: add "Copy selector:" hint in plain format | Taste | P1 — surfaces at final gate |

**Phase 3.5 complete.** DX: 4/10 initial → 7/10 with plan fixes. TTHW: ~10 min → ~4 min.

---

## Decision Audit Trail (continued)

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|-------|----------|-----------|-----------|----------|---------|
| 16 | DX | Fix scope section to match accepted decisions | Mechanical | P5 | Removes contradictions | Leave inconsistent |
| 17 | DX | README "Getting Started" section as PR2 deliverable | Mechanical | P1 | Both models: copy-paste path needed | Post-PR2 |
| 18 | DX | "hint:" line on every error message | Mechanical | P1 | Users bounce without next-step guidance | Skip |
| 19 | DX | `open_context()` return `anyhow::Result<()>` | Mechanical | P5 | Consistent with CLI error handling | Custom enum |
| 20 | DX | System launcher loops per-URL | Mechanical | P1 | `open`/`xdg-open`/`start` take one URL | Batch attempt |
| 21 | DX | CHANGELOG: 0.3.0 entry with --context + --browser deprecation | Mechanical | P1 | Trust + upgrade path | Skip |


---

## Cross-Phase Themes

**Theme 1: Firefox `--profile <absolute_path>` is a cross-crate schema change.**
Flagged independently in Phase 1 (CEO subagent), Phase 3 (Codex + Claude), and Phase 3.5 (both). High-confidence. `ProfileRef.path` field is load-bearing for correctness. Must be in PR2.

**Theme 2: `--dry-run` is critical for trust, not a nice-to-have.**
Flagged in Phase 1, Phase 3, Phase 3.5 by all voices. The macOS `open --args` silent-failure risk makes it essential. Must be in PR2.

**Theme 3: PR2 framing — "launch primitive" not "consumer product."**
Flagged in Phase 1 by both CEO voices. The PR description must say: "This PR implements the launch substrate that the rules engine will call in PR3." Not: "Now you can open URLs in the right browser." Both are true but one sets wrong expectations.

---

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | Phase 1 | Scope & strategy | 1 | issues_open | 6 decisions auto-decided; 1 user challenge resolved |
| Codex Review (CEO) | Phase 1 | Independent 2nd opinion | 1 | issues_open | 7 findings; 6 confirmed cross-model |
| Eng Review | Phase 3 | Architecture & tests | 1 | issues_open | 10 structural gaps; 9 auto-decided |
| Codex Review (Eng) | Phase 3 | Independent 2nd opinion | 1 | issues_open | 7 findings; all confirmed or improved |
| DX Review | Phase 3.5 | Developer experience | 1 | issues_open | 8 gaps; all auto-decided; score 4→7/10 |
| Codex Review (DX) | Phase 3.5 | Independent 2nd opinion | 1 | issues_open | 6 findings; all confirmed cross-model |

**VERDICT:** 22 decisions auto-decided. 0 unresolved blockers. 1 taste decision for final gate. Plan is ready to implement after approval.


---

## Final Gate — Approved

**Approval date:** 2026-04-12
**Status:** APPROVED with modifications

### User Challenge Resolved: Default-Browser Fallback

**Challenge:** `browserware-system::open_default()` (system launcher fallback for `brw open <url>` without `--context`) was added by autoplan review. User flagged: when Browserware registers as the OS default browser (PR5/6), calling `xdg-open`/`open`/`start` would create an infinite routing loop.

**Decision: REMOVED from PR2 scope.** This is architecturally wrong.

**Instead:** `brw open <url>` without `--context` prints:
```
error: no context specified
  hint: use `brw open <url> --context <selector>` to open in a specific browser context
  hint: run `brw contexts` to see available selectors
  note: rules-based routing (brw open <url> without --context) is coming in PR3
```
Exit 1. No system launcher called. No loop risk.

This is the honest, correct behavior for the current product stage.

### Taste Decision: `brw contexts` hint

`--format plain` outputs one `brw open --context <selector>` example per line. Table format unchanged.

### Final Canonical Scope

**`browserware-types`:**
- `ProfileRef.path: Option<PathBuf>` with `#[serde(default, skip_serializing_if = "Option::is_none")]`

**`browserware-profiles`:**
- Firefox parser: read `Path=` + `IsRelative=` → compute absolute profile path
- Store in `ProfileRef.path`

**`browserware-launch` (implement from stub):**
- `pub fn launch(context: &BrowserContext, urls: &[Url]) -> Result<()>`
- `pub fn build_command(context: &BrowserContext, urls: &[Url]) -> std::process::Command` (pub for testing)
- Error: `LaunchError::NotLaunchable { limitations }`, `LaunchError::ExecutableNotFound { path }`, `LaunchError::ProcessFailed { status }`, `LaunchError::EmptyUrls`
- macOS: `open -b <bundle_id> <url...> --args <profile_flags>` when `bundle_id` present
- Chromium profile: `--profile-directory=<id>` (joined, single arg)
- Firefox profile: `--profile <abs_path> --no-remote`
- Other/no profile: `exec <url...>`

**`browserware-cli` (new `commands/open.rs`, update `main.rs`):**
- `pub(crate) fn run(format: OutputFormat, context_selector: Option<&str>, urls: &[String], dry_run: bool)`
- `pub(crate) fn open_context(contexts: &[BrowserContext], selector: &str, urls: &[Url], dry_run: bool) -> anyhow::Result<()>`
- `--context <selector>`: optional selector string
- `--dry-run`: print command, exit 0, no launch
- No `--context` → print helpful error (see above), exit 1
- `AmbiguityPolicy::Error` always
- Error format: "error: {what}\n  cause: {why}\n  hint: run `brw ...`"

**`browserware-cli/src/commands/contexts.rs`:**
- Plain format: output `brw open --context <selector> <url>` hint per line (format: `brw open --context <selector>`)

**`browserware-system`:** No changes in PR2 (removed open_default — loop risk).

**`AGENTS.md`:** Remove `crates.io ✅` from architecture table.

**`README.md`:** Add "Getting Started" section with copy-paste examples.

**`CHANGELOG.md`:** Add 0.3.0 section with `--context` flag + `--browser`/`--profile` stub deprecation.

### Not in Scope (PR2)

- `browserware-system::open_default()` — deferred (loop risk with future OS registration)
- Rules engine routing — PR3
- `--on-ambiguous` flag — PR3 (hardcoded Error for now)
- crates.io publishing — TODOS.md
- GUI shim — long-term

