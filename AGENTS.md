# AGENTS.md — Instructions for AI Coding Assistants

This document provides context, guidelines, and principles for AI coding assistants
(GitHub Copilot, Claude, Cursor, etc.) working on the browserware codebase.

---

## Project Vision

**Browserware** is a cross-platform ecosystem for intelligent browser routing.

**The Problem**: Developers and power users juggle multiple browsers and profiles—
work Chrome, personal Firefox, Safari for banking. Clicking a link is a gamble:
which browser will it open in?

**The Solution**: A declarative rule engine that routes URLs to the right browser
with the right profile, every time. Cross-platform, open-source libraries and CLI,
with a premium closed-source GUI application.

**Target Users**: 
- Developers managing multiple browser profiles for different projects
- Power users separating work/personal browsing
- Security-conscious users isolating sensitive activities

---

## Licensing Model

**Open Source (MIT OR Apache-2.0)**:
- All library crates (`browserware-*`)
- CLI (`browserware-cli` / `brw`)
- Published to crates.io
- Community contributions welcome

**Closed Source (Proprietary)**:
- GUI application (private repository)
- Premium features (cloud sync, team rules)
- Never published to crates.io

This split maximizes ecosystem adoption while protecting commercial value.

---

## Core Principles

### 1. Modularity Over Monolith

Each crate has **exactly one responsibility**. If you're unsure where code belongs,
it probably deserves its own module or function.

```
✅ browserware-detect: "What browsers exist?"
✅ browserware-launch: "Run this browser with these options"

❌ browserware-detect: "What browsers exist and launch one"
```

### 2. Libraries Are Independently Useful

Every library crate should be valuable on its own, even without the CLI or GUI.
Someone should be able to `cargo add browserware-detect` for their own project.

### 3. Explicit Over Implicit

Functions receive explicit inputs rather than reaching into global state.

```rust
// ✅ Good: explicit inputs
pub fn launch(executable: &Path, family: BrowserFamily, urls: &[Url]) -> Result<()>

// ❌ Bad: implicit detection
pub fn launch(urls: &[Url]) -> Result<()>  // Where does it get the browser?
```

### 4. Fail Gracefully

Return `Result<T, Error>` rather than panicking. Errors should be descriptive
and actionable.

```rust
// ✅ Good: informative error
Error::BrowserNotFound(format!("Chrome not found at expected paths: {:?}", paths))

// ❌ Bad: vague error
Error::NotFound
```

### 5. Platform Parity

Features should work identically across macOS, Windows, and Linux unless there's
a fundamental platform limitation.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    browserware-cli (brw)                        │
│                    browserware-gui (closed)                     │
└─────────────────────────────────────────────────────────────────┘
        │           │           │           │           │
        ▼           ▼           ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  detect  │ │ profiles │ │  launch  │ │  rules   │ │  system  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
              │
              ▼
       ┌──────────┐
       │  types   │
       └──────────┘
```

### Crate Boundaries

| Crate | Single Question | Published |
|-------|-----------------|-----------|
| `browserware-types` | Shared types | crates.io ✅ |
| `browserware-detect` | "What browsers exist?" | crates.io ✅ |
| `browserware-profiles` | "What profiles exist?" | crates.io ✅ |
| `browserware-launch` | "Run this browser" | crates.io ✅ |
| `browserware-rules` | "Which browser for URL?" | crates.io ✅ |
| `browserware-system` | "OS integration" | crates.io ✅ |
| `browserware-cli` | User CLI (`brw` binary) | crates.io ✅ |
| `browserware-gui` | Desktop app | Private ❌ |

### Data Flow for `brw open https://example.com`

```
1. Parse URL
2. rules::discover_rules() → find config file
3. rules::evaluate(rules, context) → BrowserTarget
4. detect::detect_browser("chrome") → Browser
5. profiles::find_profile(family, "Work") → Profile
6. launch::launch(executable, family, urls, options) → Result
```

---

## Code Style

### Rust Conventions

- **Edition**: 2024
- **License**: MIT OR Apache-2.0 (dual license)
- **Formatting**: `rustfmt` with project `.rustfmt.toml`
- **Lints**: `clippy` with workspace lints (pedantic + nursery)

### Naming

```rust
// Types: PascalCase
pub struct BrowserFamily;
pub enum Channel;

// Functions: snake_case, verb-first for actions
pub fn detect_browsers() -> Vec<Browser>;
pub fn find_profile(name: &str) -> Option<Profile>;

// Predicates: is_/has_/can_ prefix
pub fn is_default_browser() -> bool;

// Builders: with_ prefix
pub fn with_family(self, family: BrowserFamily) -> Self;
```

### Error Handling

Use `thiserror` for library errors, `anyhow` only in the CLI binary.

```rust
// In libraries (crates/browserware-*/src/error.rs)
#[derive(Debug, Error)]
pub enum Error {
    #[error("browser not found: {0}")]
    BrowserNotFound(String),
}

// In CLI (crates/browserware-cli/src/main.rs)
fn main() -> anyhow::Result<()> {
    // anyhow for ergonomic error handling
}
```

### Documentation

Every public item needs a doc comment:

```rust
/// Detect all browsers installed on the system.
///
/// # Returns
///
/// A vector of [`Browser`] structs for each detected browser.
///
/// # Example
///
/// ```
/// let browsers = browserware_detect::detect_browsers();
/// for browser in browsers {
///     println!("{}: {}", browser.name, browser.executable.display());
/// }
/// ```
pub fn detect_browsers() -> Vec<Browser> {
    // ...
}
```

---

## Common Tasks

### Adding a New Browser

When adding support for a new browser (e.g., Arc):

1. **Update detection** (`browserware-detect/src/macos.rs`):
   ```rust
   const KNOWN_CHROMIUM_BROWSERS: &[(&str, &str)] = &[
       ("com.google.Chrome", "Google Chrome"),
       ("company.thebrowser.Browser", "Arc"),  // Add here
   ];
   ```

2. **No changes needed** in `profiles` or `launch` if it follows standard patterns

3. **Add tests**

### Adding a New Platform

1. Create `platform.rs` file (e.g., `browserware-detect/src/freebsd.rs`)
2. Add `#[cfg(target_os = "freebsd")]` module in `lib.rs`
3. Implement the same public functions
4. Add to CI matrix

### Adding a CLI Command

1. Add variant to `Commands` enum in `browserware-cli/src/main.rs`
2. Implement handler in `match` block
3. Add integration test in `tests/cli.rs`

---

## What NOT to Do

### Don't Mix Responsibilities

```rust
// ❌ Bad: detect.rs doing launching
pub fn detect_and_launch_default(url: &Url) -> Result<()> {
    let browser = detect_default()?;
    launch(&browser.executable, url)?;  // Wrong crate!
    Ok(())
}

// ✅ Good: separate concerns
// In CLI:
let browser = browserware_detect::detect_default()?;
browserware_launch::launch(&browser.executable, ...)?;
```

### Don't Assume Platform

```rust
// ❌ Bad: hardcoded path
let chrome = Path::new("/Applications/Google Chrome.app");

// ✅ Good: platform detection
#[cfg(target_os = "macos")]
fn chrome_paths() -> Vec<PathBuf> {
    vec!["/Applications/Google Chrome.app".into()]
}
```

### Don't Panic

```rust
// ❌ Bad: panicking
pub fn get_profile(name: &str) -> Profile {
    profiles.iter().find(|p| p.name == name).unwrap()
}

// ✅ Good: return Result
pub fn get_profile(name: &str) -> Result<Profile> {
    profiles.iter()
        .find(|p| p.name == name)
        .cloned()
        .ok_or_else(|| Error::ProfileNotFound(name.to_string()))
}
```

### Don't Add Unnecessary Dependencies

Before adding a dependency:
1. Is this in std?
2. Could it be a few lines of code?
3. Is it well-maintained?

---

## Platform-Specific Notes

### macOS
- Browser apps are bundles at `/Applications/*.app`
- Use `CFBundleIdentifier` for identification
- Launch Services API for default browser
- Swift shim needed for URL scheme handling

### Windows
- Registry: `HKLM\SOFTWARE\Clients\StartMenuInternet`
- `UserChoice` hash protects default browser setting
- Use `windows-rs` crate

### Linux
- XDG desktop files in `/usr/share/applications/`
- `xdg-settings` for default browser
- Multiple install locations (native, Flatpak, Snap)

---

## Commit Message Format

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): description
```

**Types**: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

**Scopes**: `types`, `detect`, `profiles`, `launch`, `rules`, `system`, `cli`

**Examples**:
```
feat(detect): add Arc browser detection on macOS
fix(launch): handle spaces in profile names
docs(readme): add installation instructions
chore(ci): update to Rust 1.82
```

---

## CLI Binary Name

**Crate name**: `browserware-cli`  
**Binary name**: `brw`

```toml
# crates/browserware-cli/Cargo.toml
[package]
name = "browserware-cli"

[[bin]]
name = "brw"
path = "src/main.rs"
```

Users install with `cargo install browserware-cli` and run with `brw`.

---

## Prior Art

Lessons from [Pathway](https://github.com/guria/pathway):
- Swift shim for macOS URL handling works well
- Chromium `Local State` JSON parsing is reliable
- `--format json` output is essential for scripting
- Infinite loop prevention is critical

---

## License

All contributions to open source crates must be compatible with MIT OR Apache-2.0.

**No license headers required in source files** — the repository-level LICENSE files apply.

---

## Summary

1. **Vision**: Route URLs to the right browser + profile automatically
2. **Licensing**: MIT/Apache open source libs + closed source GUI
3. **Architecture**: Modular crates with single responsibilities
4. **CLI**: `browserware-cli` crate → `brw` binary
5. **Style**: Rust 2024, clippy pedantic, explicit inputs, Result returns
6. **Platforms**: macOS, Windows, Linux with parity goal

When in doubt: "Would this make sense to someone reading the code for the first time?"
