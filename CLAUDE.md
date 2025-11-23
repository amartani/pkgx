# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`pkgx` is a standalone 4 MiB binary package manager/runner written in Rust. It allows running any version of any package without installing anything system-wide. The tool creates ephemeral package environments and either runs commands within those environments or injects those environments into your shell.

## Architecture

### Workspace Structure

This is a Rust workspace with two main crates:

- `crates/cli`: The CLI binary (`pkgx`)
- `crates/lib`: The core library (`libpkgx` v0.7.0) - provides the installtion and execution primitives

### Core Concepts

**Package Environment Creation**: Everything `pkgx` does involves creating package environments using the `+pkg` syntax. For example, `pkgx +node -- node start` creates an environment with Node.js and its dependencies, then runs `node start` within that environment.

**The Pantry**: Package metadata repository that contains:
- Package definitions in `package.yml` files
- Dependencies, provided programs, companions, and runtime environment variables
- Platform-specific configuration (darwin/linux/windows, aarch64/x86-64)

**The Cellar**: Local package storage at `~/.pkgx/<project>/<version>/` (configurable via `PKGX_DIR` or `XDG_DATA_HOME`)

**Distribution**: Packages are fetched from `dist.pkgx.dev` as prebuilt "bottles" (tar.gz/tar.xz archives) organized by:
```
dist.pkgx.dev/<project>/<platform>/<arch>/
├── versions.txt
├── v<VERSION>.tar.{gz,xz}
├── v<VERSION>.sha256sum
└── v<VERSION>.asc
```

### Key Library Modules

**`crates/lib/src/hydrate.rs`**: Dependency resolution and topological sorting
- Recursively resolves dependencies using a graph-based approach
- Intersects version constraints to find compatible versions
- Returns a topologically sorted list of packages
- **NOTE**: Optimization efforts should focus here first

**`crates/lib/src/resolve.rs`**: Package resolution
- Checks if packages are already installed locally (cellar)
- Falls back to remote inventory if not found locally
- Uses concurrent futures for parallel resolution

**`crates/lib/src/install.rs`**: Package installation
- Downloads packages from `dist.pkgx.dev`
- Uses file locking to prevent concurrent installation conflicts
- Streams and extracts tar archives (gzip/xz compressed)
- Verifies packages before finalizing installation

**`crates/lib/src/cellar.rs`**: Local package management
- Lists installed packages for a given project
- Resolves version constraints against locally installed packages
- Manages the `~/.pkgx` directory structure

**`crates/lib/src/pantry.rs`**: Pantry metadata parsing
- Iterates through `package.yml` files in the pantry
- Deserializes package definitions with platform-specific handling
- Extracts dependencies, programs, companions, and environment variables

**`crates/lib/src/config.rs`**: Configuration management
- `pantry_dir`: Defaults to `~/.local/share/pkgx/pantry` (or `PKGX_PANTRY_DIR`)
- `pantry_db_file`: SQLite database at `~/.cache/pkgx/pantry.2.db`
- `dist_url`: Package distribution URL (defaults to https://dist.pkgx.dev or `PKGX_DIST_URL`)
- `pkgx_dir`: Package installation directory (defaults to `~/.pkgx` or `PKGX_DIR`)

**`crates/lib/src/sync.rs`**: Pantry synchronization
- Syncs pantry metadata from the remote repository
- Updates the local SQLite database with package information

**`crates/lib/src/env.rs`**: Environment variable generation
- Constructs `PATH`, `MANPATH`, `PKG_CONFIG_PATH`, `LIBRARY_PATH`, etc.
- Handles platform-specific variables like `DYLD_FALLBACK_LIBRARY_PATH` (macOS)

### CLI Entry Points

**`crates/cli/src/main.rs`**: Main entry point
- Parses command-line arguments
- Handles three modes: Help, Version, Query, and X (execute)
- Sets up configuration, database connection, and spinner
- Manages pantry sync

**`crates/cli/src/args.rs`**: Argument parsing

**`crates/cli/src/execve.rs`**: Process execution via `execve()`

**`crates/cli/src/query.rs`**: Query mode (`pkgx -Q`) - lists available packages

**`crates/cli/src/x.rs`**: Execute mode - finds programs and prepares execution

**`crates/cli/src/resolve.rs`**: CLI-level resolution that orchestrates lib resolution and hydration

## Development Commands

### Building

```bash
# Debug build
cargo build

# Release build (optimized, recommended for testing)
cargo build --release

# The binary will be at target/{debug,release}/pkgx
```

### Testing

```bash
# Run all tests
cargo test --all-features

# Run tests for a specific crate
cargo test -p libpkgx
cargo test -p pkgx
```

### Linting

```bash
# Format check
cargo fmt --all --check

# Auto-format
cargo fmt --all

# Clippy linting
cargo clippy --all-features

# Markdown linting (requires npx)
npx markdownlint --config .github/markdownlint.yml --fix .
```

### Running

```bash
# After building
./target/debug/pkgx +node -- node --version

# Or directly with cargo
cargo run -- +git git --version

# Query available packages
cargo run -- -Q git

# Show environment for a package
cargo run -- +python
```

## Cache and Pantry Structure

### Local Cache Structure

When you run `pkgx`, it creates the following structure:

```
~/.pkgx/                          # The cellar (installation directory)
├── <project>/                    # e.g., nodejs.org/
│   ├── v<version>/               # e.g., v20.5.0/
│   │   ├── bin/
│   │   ├── lib/
│   │   ├── include/
│   │   └── share/
│   └── var/                      # Volatile data (excluded from versioning)

~/.cache/pkgx/                    # Cache directory
└── pantry.2.db                   # SQLite database of pantry metadata

~/.local/share/pkgx/pantry/       # Synced pantry metadata
└── projects/                     # Package definitions
    └── <org>/<project>/          # e.g., nodejs.org/
        └── package.yml           # Package metadata
```

### Pantry API Structure

The pantry at `dist.pkgx.dev` provides:

**Bottles (prebuilt binaries)**:
- `dist.pkgx.dev/<project>/<platform>/<arch>/versions.txt`: Available versions
- `dist.pkgx.dev/<project>/<platform>/<arch>/v<version>.tar.{gz,xz}`: Package archive
- `dist.pkgx.dev/<project>/<platform>/<arch>/v<version>.sha256sum`: Checksum
- `dist.pkgx.dev/<project>/<platform>/<arch>/v<version>.asc`: Signature

Example: `dist.pkgx.dev/nodejs.org/linux/x86-64/v20.5.0.tar.xz`

**Source mirrors**:
- `dist.pkgx.dev/<project>/versions.txt`
- `dist.pkgx.dev/<project>/v<version>.tar.gz`

Note: Source versions.txt may differ from bottle versions.txt—always check the platform-specific versions.

### Package.yml Structure

Pantry `package.yml` files contain:
- `dependencies`: Runtime dependencies with version constraints
- `provides`: List of programs this package provides (platform-specific support via maps)
- `companions`: Optional packages that work well together
- `runtime.env`: Environment variables (platform and arch-specific support)
- `display-name`: Optional human-readable name

## Common Patterns

### Adding a New Package

Packages are defined in the separate [pantry repository](https://github.com/pkgxdev/pantry). This repository only handles the runner/installer logic.

### Error Handling

The codebase uses `Result<T, Box<dyn Error>>` extensively for error propagation. Custom error types are defined where needed (e.g., `ResolveError` in `crates/lib/src/resolve.rs`).

### Async/Await

Heavy use of Tokio for async operations:
- HTTP requests (reqwest)
- File I/O (tokio::fs)
- Concurrent package resolution (FuturesUnordered)
- Streaming downloads with async-compression

### Platform Handling

Platform-specific code uses Rust's `cfg` attributes:
```rust
#[cfg(target_os = "macos")]
#[cfg(target_os = "linux")]
#[cfg(target_os = "windows")]
#[cfg(target_arch = "aarch64")]
#[cfg(target_arch = "x86_64")]
```

## Important Notes

### Standalone Binary

The binary is designed to be completely standalone:
- On non-macOS: OpenSSL is vendored (`features = ["vendored"]`)
- SQLite is bundled on non-macOS (`features = ["bundled"]`)
- macOS uses system-provided OpenSSL and SQLite

### Locking Mechanism

Installation uses file-based locking (`fs2::FileExt::lock_exclusive`) to prevent race conditions when multiple `pkgx` processes try to install the same package.

### Environment Isolation

`pkgx` does not pollute the system:
- Packages are downloaded to `~/.pkgx` but the wider system is untouched
- No global modifications to shell configuration (unless using the separate `dev` tool)
- Each invocation creates an isolated environment

### Performance Considerations

- Dependency resolution (hydrate.rs) is the primary optimization target
- Concurrent resolution and installation where possible
- Streaming decompression to avoid memory bloat
- Local cellar cache prevents redundant downloads

## Testing in CI

Integration tests run actual package installations:
- Install real packages (e.g., `gnome.org/libxml2`)
- Test concurrent installations to ensure locking works
- Verify environment variable generation
- Test various invocation patterns (`pkgx +git`, `pkgx git --version`, etc.)
- Validate `--sync` updates pantry correctly
- Coverage is tracked via Coveralls (unit + integration)

## Related Projects

- `dev`: Separate tool for project-specific environments using `pkgx` primitives
- `pkgm`: Installs packages globally to `/usr/local`
- `mash`: Script package manager built on top of `pkgx`
- The [pantry](https://github.com/pkgxdev/pantry): Package definitions repository

## Migration Notes (v1 to v2)

v2 tightened scopes:
- Removed shellcode integration (now in separate `dev` tool)
- Removed `env +foo` mode (use `eval "$(pkgx +foo)"` instead)
- Replaced `pkgx install` with separate `pkgm` tool
