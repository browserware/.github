# Browserware

**Browserware** is a modular ecosystem for **intelligent browser routing, discovery, and control** across macOS, Windows, and Linux.

It is designed for developers and power users who want **deterministic, scriptable, and inspectable control** over how URLs are opened: which browser, which profile, and under which conditions.

Browserware is **not a browser**.  
It is infrastructure for working *with* browsers.

---

## What Browserware Is

- A **set of small, focused Rust libraries** with clear responsibilities
- A **CLI (`brw`)** that orchestrates those libraries
- A **future GUI** for visual rule editing and browser/profile selection (closed source)
- A **system integration layer** to act as a default browser safely

Each component is useful on its own, but together they form a complete browser-routing platform.

---

## What Browserware Is Not

- ❌ Not a browser
- ❌ Not a browser extension
- ❌ Not a monolithic app

Browserware favors **local-first**, **transparent logic**, and **explicit user control**.

---

## Repository Layout

This repository holds organization-wide documentation for Browserware.

### Main Codebase

The actual code and implementation lives in the monorepo:

`https://github.com/browserware/browserware`

Inside the monorepo:

```
crates/
├── browserware-types     # Shared enums, IDs, errors
├── browserware-detect    # Installed browser discovery
├── browserware-profiles  # Profile enumeration & temp profiles
├── browserware-launch    # Cross-browser launch engine
├── browserware-rules     # Pure routing & rule evaluation
├── browserware-system    # OS integration (default browser, handlers)
└── browserware-cli       # CLI application (brw binary)
```

Other repositories:
- **GUI application**: Private repository (closed source)
- Future: Documentation site, integrations

---

## Core Libraries

### `browserware-detect`
Detects installed browsers and their metadata:
- Browser family (Chromium, Firefox, WebKit, Other)
- Channel (stable, beta, dev, canary, nightly)
- Executable paths and identifiers

### `browserware-profiles`
Manages browser profiles:
- Enumerate existing profiles
- Resolve profile data directories
- Create ephemeral / temporary profiles

### `browserware-launch`
Launches browsers reliably:
- Correct CLI flags per browser family
- Profile selection
- Incognito, new window, kiosk modes
- Inspectable command construction

### `browserware-rules`
Pure routing engine:
- TOML / JSON rule parsing
- Domain, path, regex, glob matching
- Priority and fallback resolution
- No OS or browser dependencies

### `browserware-system`
Platform integration:
- Register / unregister as default browser
- Receive URLs from the OS
- Detect source application
- Prevent routing loops

---

## CLI: `brw`

The CLI is the primary user-facing entry point.

Examples:

```bash
# Install via cargo
cargo install browserware-cli

# List detected browsers
brw browsers

# Open URL with routing
brw open https://github.com

# Open with specific browser and profile
brw open --browser chrome --profile "Work" https://slack.com

# Register as default browser
brw register
```

The CLI is intentionally thin:  
**all logic lives in libraries**.

---

## Design Principles

1. **Single Responsibility**  
   Each crate does one thing well.

2. **Independent Utility**  
   Libraries are usable standalone.

3. **No Hidden Coupling**  
   Explicit inputs, explicit outputs.

4. **Platform Abstraction**  
   Unified APIs with OS-specific implementations.

5. **Inspectability**  
   Commands and decisions can be inspected and debugged.

---

## Licensing

Browserware uses a split licensing model:

### Open Source (MIT OR Apache-2.0)

All library crates and the CLI are open source:
- `browserware-types`
- `browserware-detect`
- `browserware-profiles`
- `browserware-launch`
- `browserware-rules`
- `browserware-system`
- `browserware-cli` (brw binary)

**This maximizes ecosystem adoption** — anyone can use, fork, and contribute to these libraries.

### Closed Source (Proprietary)

The GUI application is closed source:
- Private repository
- Premium features (cloud sync, team rules)
- Permanent protection
- Not published to crates.io

**This protects commercial value** — the product features that monetize the project are proprietary.

For more details on the licensing strategy, see the main repository documentation.

---

## Status

🚧 **Early development**

- APIs may change
- Expect breaking changes before `v1.0`
- Feedback and design discussion are welcome

---

## Contributing

Contributions are welcome under the terms of the license.

See:
- `CONTRIBUTING.md`
- `SECURITY.md`
- `AGENTS.md` (for AI coding assistants)

Please keep contributions:
- Small
- Focused
- Aligned with the library boundaries

---

## Philosophy

Browsers are powerful.  
Profiles are messy.  
Defaults are opinionated.

Browserware exists to make browser behavior:
- **Predictable**
- **Composable**
- **Automatable**

Nothing more. Nothing less.
