# pkgx API and Data Structures

This document describes the public interfaces and data structures for clients that wish to interact with pkgx infrastructure, either via HTTP APIs or by directly accessing local storage.

## Table of Contents

1. [Distribution API (HTTP)](#distribution-api-http)
2. [Local Storage Structure](#local-storage-structure)
3. [Pantry Metadata Format](#pantry-metadata-format)
4. [SQLite Database Schema](#sqlite-database-schema)
5. [Version Specifications](#version-specifications)
6. [Platform and Architecture Identifiers](#platform-and-architecture-identifiers)

---

## Distribution API (HTTP)

The primary distribution server is `https://dist.pkgx.dev`. This can be overridden via the `PKGX_DIST_URL` environment variable.

### Bottle Distribution (Prebuilt Binaries)

#### Get Available Versions

```
GET {DIST_URL}/{project}/{platform}/{arch}/versions.txt
```

**Example:**
```
GET https://dist.pkgx.dev/nodejs.org/linux/x86-64/versions.txt
```

**Response Format:**
```
14.20.1
14.21.0
14.21.1
16.19.0
16.20.0
18.14.0
20.5.0
```

Newline-separated list of version numbers, sorted.

#### Download Package

```
GET {DIST_URL}/{project}/{platform}/{arch}/v{version}.tar.{gz|xz}
```

**Example:**
```
GET https://dist.pkgx.dev/nodejs.org/linux/x86-64/v20.5.0.tar.xz
```

**Response:** Compressed tar archive containing the package.

**Note:** `.tar.xz` format is preferred over `.tar.gz` when available.

#### Verify Package Integrity

**SHA256 Checksum:**
```
GET {DIST_URL}/{project}/{platform}/{arch}/v{version}.sha256sum
```

**GPG Signature:**
```
GET {DIST_URL}/{project}/{platform}/{arch}/v{version}.asc
```

### Source Distribution (Mirrors)

#### Get Available Source Versions

```
GET {DIST_URL}/{project}/versions.txt
```

**Note:** Source `versions.txt` may differ from bottle `versions.txt`. Always use the platform-specific versions when installing prebuilt binaries.

#### Download Source Archive

```
GET {DIST_URL}/{project}/v{version}.tar.gz
```

---

## Local Storage Structure

### Directory Layout

pkgx uses three primary directories:

#### 1. The Cellar (Package Installations)

**Default Location:** `~/.pkgx/` (or `$PKGX_DIR`, or `$XDG_DATA_HOME/pkgx/`)

```
~/.pkgx/
├── {project}/                      # e.g., nodejs.org/
│   ├── v{version}/                 # e.g., v20.5.0/
│   │   ├── bin/                    # Executable binaries
│   │   ├── lib/                    # Shared libraries
│   │   ├── include/                # Header files
│   │   ├── share/                  # Shared data files
│   │   │   ├── man/                # Manual pages
│   │   │   └── doc/                # Documentation
│   │   └── ...                     # Other standard directories
│   ├── v{another_version}/
│   └── var/                        # Volatile data (not versioned)
└── {another_project}/
```

**Path Pattern:**
```
{PKGX_DIR}/{project}/v{version}/
```

**Example:**
```
~/.pkgx/nodejs.org/v20.5.0/bin/node
```

#### 2. The Cache Directory

**Default Location:** `~/.cache/pkgx/`

```
~/.cache/pkgx/
└── pantry.2.db                     # SQLite database (see schema below)
```

#### 3. The Pantry Directory

**Default Location:** `~/.local/share/pkgx/pantry/` (or `$PKGX_PANTRY_DIR`)

```
~/.local/share/pkgx/pantry/
└── projects/                       # Package definitions
    ├── {org}/                      # e.g., nodejs.org/
    │   └── package.yml             # Package metadata
    ├── {another_org}/
    │   └── {project}/
    │       └── package.yml
    └── ...
```

### Environment Variable Configuration

| Variable | Purpose | Default |
|----------|---------|---------|
| `PKGX_DIR` | Package installation directory | `~/.pkgx` or `$XDG_DATA_HOME/pkgx` |
| `PKGX_PANTRY_DIR` | Pantry metadata directory | `~/.local/share/pkgx/pantry` |
| `PKGX_DIST_URL` | Distribution server URL | `https://dist.pkgx.dev` |

**Note:** If `PKGX_PANTRY_DIR` is set, the SQLite database will be located at `{PKGX_PANTRY_DIR}/pantry.2.db` instead of the cache directory.

---

## Pantry Metadata Format

Package metadata is stored in YAML format at `projects/{project}/package.yml`.

### Schema

```yaml
# Optional: Human-readable display name
display-name: Node.js

# Runtime dependencies with version constraints
dependencies:
  zlib.net: ^1.2
  openssl.org: ^1.1
  # Platform-specific dependencies
  linux:
    gnu.org/libc: ^2.28
  darwin:
    apple.com/xcode/clt: '*'

# Programs provided by this package
provides:
  - bin/node
  - bin/npm
  - bin/npx
  # Or platform-specific:
  # darwin:
  #   - bin/node
  #   - bin/npm

# Optional packages that work well together
companions:
  yarnpkg.com: ^1.22

# Runtime environment variables
runtime:
  env:
    NODE_PATH: "{{prefix}}/lib/node_modules"
    # Platform-specific environment
    darwin:
      DYLD_FALLBACK_LIBRARY_PATH: "{{prefix}}/lib"
    linux:
      LD_LIBRARY_PATH: "{{prefix}}/lib"
    # Architecture-specific environment
    aarch64:
      ARCH_SPECIFIC_VAR: value
    x86-64:
      ARCH_SPECIFIC_VAR: value
```

### Field Descriptions

#### `dependencies`
Maps project names to version constraints. Supports platform-specific overrides via nested maps with keys: `darwin`, `linux`, `windows`.

**Version Constraint Format:**
- `^1.2`: Compatible with 1.2.x (>= 1.2.0, < 2.0.0)
- `~1.2`: Approximately 1.2.x (>= 1.2.0, < 1.3.0)
- `1.2`: Exact version match (numeric values are prefixed with ^)
- `*`: Any version
- `>=1.2`: Greater than or equal to 1.2
- `<2.0`: Less than 2.0

#### `provides`
List of programs this package provides. Can be:
- Simple list of program paths
- Platform-specific map with keys: `darwin`, `linux`, `windows`

Programs are indexed by their basename (e.g., `bin/node` → indexed as `node`).

#### `companions`
Optional packages that enhance functionality. Same format as `dependencies`.

#### `runtime.env`
Environment variables set when this package is in use. Supports:
- Top-level key-value pairs
- Platform-specific nested maps (`darwin`, `linux`, `windows`)
- Architecture-specific nested maps (`aarch64`, `x86-64`)

**Template Variables:**
- `{{prefix}}`: Replaced with the package installation directory

---

## SQLite Database Schema

The pantry database (`pantry.2.db`) caches parsed metadata for fast lookups.

### Tables

#### `provides`
Maps programs to projects.

```sql
CREATE TABLE provides (
    project TEXT,
    program TEXT
);
CREATE INDEX idx_project ON provides(project);
CREATE INDEX idx_program ON provides(program);
```

**Example Rows:**
```
project         | program
----------------|--------
nodejs.org      | node
nodejs.org      | npm
git-scm.org     | git
```

#### `dependencies`
Package dependencies.

```sql
CREATE TABLE dependencies (
    project TEXT,
    pkgspec TEXT
);
CREATE INDEX idx_project_dependencies ON dependencies(project);
```

**Example Rows:**
```
project         | pkgspec
----------------|------------------
nodejs.org      | zlib.net^1.2
nodejs.org      | openssl.org^1.1
```

#### `companions`
Optional companion packages.

```sql
CREATE TABLE companions (
    project TEXT,
    pkgspec TEXT
);
CREATE INDEX idx_project_companions ON companions(project);
```

#### `runtime_env`
Runtime environment variables.

```sql
CREATE TABLE runtime_env (
    project TEXT,
    envline TEXT
);
```

**Format:** `envline` is stored as `KEY=VALUE`.

**Example Rows:**
```
project         | envline
----------------|-----------------------------------
nodejs.org      | NODE_PATH={{prefix}}/lib/node_modules
```

#### `aliases`
Display name aliases.

```sql
CREATE TABLE aliases (
    project TEXT,
    alias TEXT
);
CREATE INDEX idx_alias_project ON aliases(alias);
```

**Example Rows:**
```
project         | alias
----------------|--------
nodejs.org      | Node.js
python.org      | Python
```

### Common Queries

**Find which package provides a program:**
```sql
SELECT project FROM provides WHERE program = 'node';
```

**Get dependencies for a project:**
```sql
SELECT pkgspec FROM dependencies WHERE project = 'nodejs.org';
```

**Find package by display name:**
```sql
SELECT project FROM aliases WHERE LOWER(alias) = LOWER('Node.js');
```

**Get runtime environment for a project:**
```sql
SELECT envline FROM runtime_env WHERE project = 'nodejs.org';
```

---

## Version Specifications

pkgx uses semantic versioning with constraint syntax compatible with `libsemverator`.

### Package Specification Format

**Pattern:** `{project}[{constraint}]`

**Examples:**
- `nodejs.org` → Any version (equivalent to `nodejs.org*`)
- `nodejs.org@20` → Version 20.x.x (prefix @ means exact match start)
- `nodejs.org^20.5` → Compatible with 20.5.x (>= 20.5.0, < 21.0.0)
- `nodejs.org~20.5` → Approximately 20.5.x (>= 20.5.0, < 20.6.0)
- `nodejs.org=20.5.0` → Exact version 20.5.0
- `nodejs.org>=20.5` → Greater than or equal to 20.5
- `nodejs.org<21` → Less than 21.0.0

### Constraint Parsing

The package specification regex: `^(.+?)(([\^=~<>@].+)|\*)?$`

1. **Capture Group 1:** Project name
2. **Capture Group 2:** Version constraint (optional)

If no constraint is provided, defaults to `*` (any version).

### Version Resolution

When resolving versions:
1. Fetch available versions from `{DIST_URL}/{project}/{platform}/{arch}/versions.txt`
2. Parse each line as a semantic version
3. Filter versions that satisfy the constraint
4. Select the maximum (latest) matching version

---

## Platform and Architecture Identifiers

### Platform (OS)

| Value | Operating System |
|-------|------------------|
| `darwin` | macOS |
| `linux` | Linux |
| `windows` | Windows |

### Architecture

| Value | CPU Architecture |
|-------|------------------|
| `aarch64` | ARM 64-bit (Apple Silicon, ARM servers) |
| `x86-64` | x86 64-bit (Intel/AMD) |

### Detection

Platform and architecture are detected at compile time:
- Platform: `target_os` (macos → darwin, linux → linux, windows → windows)
- Architecture: `target_arch` (aarch64 → aarch64, x86_64 → x86-64)

### URL Construction

```
{DIST_URL}/{project}/{platform}/{arch}/v{version}.tar.xz
```

**Example:**
```
https://dist.pkgx.dev/nodejs.org/darwin/aarch64/v20.5.0.tar.xz
```

---

## Package Archive Format

Downloaded packages are tar archives (gzip or xz compressed) containing a directory structure ready to be extracted into `{PKGX_DIR}/{project}/v{version}/`.

### Archive Structure

```
v{version}/
├── bin/
│   └── {executables}
├── lib/
│   └── {libraries}
├── include/
│   └── {headers}
├── share/
│   ├── man/
│   └── doc/
└── ...
```

### Extraction

1. Download archive to temporary location
2. Acquire exclusive lock on `{PKGX_DIR}/{project}/` (or `{PKGX_DIR}/{project}/lockfile` on Windows)
3. Check if `{PKGX_DIR}/{project}/v{version}/` already exists (another process may have installed it)
4. Extract archive to temporary directory
5. Move to final location: `{PKGX_DIR}/{project}/v{version}/`
6. Release lock

**Concurrency:** File locking ensures multiple processes can safely attempt to install the same package simultaneously. Only one will perform the installation; others will wait and use the result.

---

## Environment Variable Generation

When executing programs, pkgx constructs environment variables from installed packages:

### Standard Variables

- `PATH`: Prepends `{prefix}/bin` for each package
- `MANPATH`: Prepends `{prefix}/share/man`
- `PKG_CONFIG_PATH`: Prepends `{prefix}/lib/pkgconfig`
- `LIBRARY_PATH`: Prepends `{prefix}/lib`
- `CPATH`: Prepends `{prefix}/include`
- `XDG_DATA_DIRS`: Prepends `{prefix}/share`

### Platform-Specific Variables

**macOS:**
- `DYLD_FALLBACK_LIBRARY_PATH`: Prepends `{prefix}/lib`

**Linux:**
- `LD_LIBRARY_PATH`: Prepends `{prefix}/lib`

### Package-Specific Variables

Additional variables from `runtime.env` in `package.yml`, with `{{prefix}}` replaced by the installation path.

---

## Client Implementation Guidelines

### Recommended Workflow

1. **Query Package:**
   - Fetch `{DIST_URL}/{project}/{platform}/{arch}/versions.txt`
   - Parse and filter by version constraint
   - Select the latest matching version

2. **Check Local Cache:**
   - Check if `{PKGX_DIR}/{project}/v{version}/` exists
   - If yes, use it directly

3. **Download and Install:**
   - Download `{DIST_URL}/{project}/{platform}/{arch}/v{version}.tar.xz`
   - Verify checksum from `.sha256sum` file (optional but recommended)
   - Acquire file lock on package directory
   - Extract to installation location
   - Release lock

4. **Resolve Dependencies:**
   - Query SQLite database or parse `package.yml` for dependencies
   - Recursively resolve and install dependencies
   - Build topologically sorted list (dependencies before dependents)

5. **Construct Environment:**
   - For each package in dependency order:
     - Add to `PATH`, `MANPATH`, etc.
     - Apply package-specific environment variables
   - Execute target program with constructed environment

### Pantry Synchronization

The pantry metadata should be periodically synchronized:

1. Clone or pull from `https://github.com/pkgxdev/pantry`
2. Extract to `{PKGX_PANTRY_DIR}/`
3. Rebuild SQLite database by parsing all `package.yml` files

**Note:** pkgx itself performs this sync automatically when needed, or when explicitly requested with `--sync`.

---

## Error Handling

### Common Error Scenarios

**Package Not Found:**
- HTTP 404 when fetching `versions.txt`
- Empty `versions.txt`

**No Matching Version:**
- No versions satisfy the constraint after filtering

**Download Failures:**
- HTTP errors during download
- Checksum mismatch

**Installation Conflicts:**
- Handled via file locking (automatic retry/wait)

### HTTP Response Codes

- `200 OK`: Successful request
- `404 Not Found`: Package/version not available
- `403 Forbidden`: Access denied (rare, usually network issues)

---

## Security Considerations

1. **HTTPS:** Always use HTTPS for distribution URLs to prevent MITM attacks
2. **Checksum Verification:** Verify `.sha256sum` after download
3. **Signature Verification:** Optionally verify `.asc` GPG signatures
4. **File Permissions:** Ensure proper permissions on installed files
5. **Symlink Safety:** Be cautious when extracting archives with symlinks

---

## Additional Resources

- **Pantry Repository:** https://github.com/pkgxdev/pantry
- **Package Listings:** https://pkgx.dev/pkgs/
- **Documentation:** https://docs.pkgx.sh
- **Distribution Index:** https://dist.pkgx.dev (browse via web)

---

## Changelog

**v2 (Current):**
- Database file: `pantry.2.db`
- Archive format: `.tar.xz` preferred over `.tar.gz`
- Platform identifiers: `darwin`, `linux`, `windows`
- Architecture identifiers: `aarch64`, `x86-64`

**Future Considerations:**
- The database schema and API format may evolve; version indicators in filenames (`pantry.2.db`) help identify compatibility
