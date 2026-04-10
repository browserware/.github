# TODOs

## Release Automation

### Add crates.io publishing workflow for Browserware crates

**What:** Add release automation that publishes `browserware-cli` and the Browserware library crates to crates.io.

**Why:** The design and README install path rely on `cargo install browserware-cli`, but the existing GitHub Actions release workflow only builds GitHub Release binaries.

**Pros:**
- Makes the documented `cargo install browserware-cli` path real.
- Reduces manual release toil.
- Keeps CLI and library distribution aligned with the Rust ecosystem.

**Cons:**
- Requires crates.io token/secrets setup.
- Requires careful workspace crate publish ordering.
- Adds another release path to maintain alongside GitHub Release binaries.

**Context:** Existing `.github/workflows/release.yml` builds cross-platform `brw` binaries and uploads them to GitHub Releases. It does not publish crates to crates.io. The Browser Context Substrate PR intentionally defers this so PR1 can focus on `BrowserContext` and `brw contexts`.

**Depends on / blocked by:** Not required for PR1. Should be completed before documentation broadly advertises `cargo install browserware-cli` as the primary install path.
