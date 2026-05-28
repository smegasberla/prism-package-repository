# Prism Package Registry

The **Prism Package Registry** is a fully automated, serverless package repository powered by GitHub.
There is no central server, no database, and no manual review process. Every step — from publishing
a package to resolving it by name — runs through GitHub's API, Git, and a single GitHub Action.

## Architecture at a Glance

```
┌──────────────┐     fork + push PR      ┌──────────────────────────┐
│  prism       │ ────────────────────────▶│  prism-package-repository │
│  publish     │                          │  (this repo)              │
└──────────────┘                          │                           │
                                          │  .github/workflows/       │
                                          │    publish.yml  ◀─────────┤  GitHub Action
                                          │                           │  (auto-merge)
                                          │  index/                   │
                                          │    1/name                 │
                                          │    2/ab                  │
                                          │    3/a/abc               │
                                          │    ab/cd/abcd            │
                                          └──────────────────────────┘
                                                    │
                                                    │ git pull
                                                    ▼
┌──────────────┐     resolves path       ┌──────────────────────────┐
│  prism       │ ◀───────────────────────│  ~/.prism/registry/       │
│  install     │     git clone           │  (local mirror)           │
└──────────────┘                          └──────────────────────────┘
                                                    │
                                                    │ git clone
                                                    ▼
                                          ┌──────────────────────────┐
                                          │  ~/.local/lib/prism/      │
                                          │    packages/<name>/       │
                                          └──────────────────────────┘
```

**Key insight:** There is no server. The "registry" is just a Git repository with NDJSON index files.
GitHub Actions provide automation, and `prism install` clones package source repos directly.

## The Registry Index

The registry stores metadata as **NDJSON** (newline-delimited JSON) files in a directory tree.
Each package gets one file; each version is one line in that file.

### Path Algorithm (Cargo-style)

The index path for a package is computed from its name, using the same algorithm as Cargo:

| Name length | Path pattern     | Example        | File path              |
|-------------|------------------|----------------|------------------------|
| 1           | `index/1/<name>` | `a`            | `index/1/a`            |
| 2           | `index/2/<name>` | `ab`           | `index/2/ab`           |
| 3           | `index/3/<c>/<name>` | `abc`     | `index/3/a/abc`        |
| 4+          | `index/<0-1>/<2-3>/<name>` | `abcd` | `index/ab/cd/abcd` |

This algorithm is implemented identically in:
- **C** (`src/package/package.c` → `registry_path_for_package()`) — used by `prism publish` and `prism install`
- **Python** (`.github/workflows/publish.yml`) — used by the GitHub Action that processes PRs

### NDJSON File Format

Each line is a complete JSON object:

```json
{"name":"http","vers":"0.1.0","deps":["json"]}
```

**Fields:**
| Field    | Required | Description                                      |
|----------|----------|--------------------------------------------------|
| `name`   | ✅       | Package name (alphanumeric, hyphens, underscores, dots) |
| `vers`   | ✅       | Semantic version (e.g. `0.1.0`, `1.2.3-beta`)   |
| `deps`   | ❌       | Array of dependency package names                 |

Multiple versions of the same package are represented as multiple lines:

```
{"name":"http","vers":"0.1.0","deps":["json"]}
{"name":"http","vers":"0.2.0","deps":["json","net"]}
```

## How Publishing Works

When a package author runs `prism publish`:

### 1. CLI (`prism publish`)
- Reads and validates `prism-package.json` (name, version, repo fields)
- Computes the registry index path using the Cargo-style algorithm
- Uses the GitHub API to fork `prism-package-repository`
- Creates a branch in the fork with the NDJSON metadata file
- Opens a pull request **whose body is the JSON payload**:
  ```json
  {"name":"http","version":"0.1.0","description":"HTTP library","repo":"https://github.com/...","dependencies":["json"]}
  ```
- The CLI never pushes source code to the registry — only metadata.

### 2. GitHub Action (`.github/workflows/publish.yml`)
- **Triggers on:** `pull_request_target` → `opened`, `reopened`
- **Security:** Runs in the context of the registry repo (not the fork). Never checks out fork code — only reads the PR body (plain text).
- **Steps:**
  1. Parses the JSON from the PR body
  2. Validates every field (name, version, repo URL against regex)
  3. Computes the index path (same algorithm as CLI)
  4. Creates parent directories if needed
  5. Appends the version as one NDJSON line to the index file
  6. Commits directly to `main`
  7. Squash-merges the PR (cleanly closes it)

### 3. The package is now published
Anyone can install it with `prism install <name>`.

## Security Model

The registry is designed with defense-in-depth security:

| Layer | Mechanism | What it prevents |
|-------|-----------|------------------|
| **CLI validation** | Name/version/repo regex checks in C | Malformed metadata from reaching the registry |
| **Action validation** | Same regex checks in Python (defense in depth) | Malicious PR bodies that bypass the CLI |
| **`pull_request_target`** | Action runs with registry's own token, not the fork's | Fork code execution; token exfiltration |
| **PR body only** | Action never checks out fork code | Supply-chain attacks via malicious source |
| **No shell injection** | CLI sends JSON in PR body, file writes use `printf` | Command injection via package metadata |
| **GIT_ASKPASS** | Token passed via env, never in command line | Token leakage via `ps` or shell history |

## How Installation Works

When a user runs `prism install <name>`:

1. **Sync the registry index** — `git pull` in `~/.prism/registry/` to get the latest metadata
2. **Resolve the package** — Read `index/<path>/<name>` → parse NDJSON → find the latest version
3. **Clone the source** — Parse the `repo` field from the published PR body metadata, then `git clone` into `~/.local/lib/prism/packages/<name>/`
4. **The package is ready** — `prism add <name>` registers it in your project

The local registry mirror (`~/.prism/registry/`) is a full Git clone of `prism-package-repository`.
This means `prism install` doesn't need a GitHub token — only `publish` and `registry init` do.

## Bootstrapping the Ecosystem

The entire registry infrastructure can be created with a single command:

```bash
export PRISM_GITHUB_TOKEN=ghp_yourTokenHere
prism registry init
```

This command:
1. Creates the `prism-package-repository` GitHub repo
2. Pushes the `publish.yml` workflow to `.github/workflows/`
3. Configures Actions permissions (read+write, allow PR approval)
4. Initializes the `index/` directory structure
5. Commits and pushes everything

After this, anyone can publish packages and anyone can install them — zero manual setup.

## Manual Setup (Alternative)

If you prefer to set up the registry manually:

1. Create a public GitHub repo named `prism-package-repository`
2. Copy `publish.yml` from the Prism source (or `~/.local/lib/prism/registry/publish.yml` after installing Prism) to `.github/workflows/publish.yml`
3. In repo Settings → Actions → General, set **Read and write permissions** and enable **Allow GitHub Actions to create and approve pull requests**
4. Create the initial index structure:
   ```bash
   mkdir -p index/1 index/2 index/3 index/aa
   git add index/ && git commit -m "init" && git push
   ```

## Repository Structure

```
prism-package-repository/
├── .github/
│   └── workflows/
│       └── publish.yml          # Auto-merge GitHub Action
├── index/
│   ├── 1/                       # 1-character packages
│   │   └── a                    # (e.g. package "a")
│   ├── 2/                       # 2-character packages
│   │   └── ab                   # (e.g. package "ab")
│   ├── 3/                       # 3-character packages
│   │   ├── a/
│   │   │   └── abc
│   │   └── b/
│   │       └── bar
│   ├── ab/                      # 4+ character packages
│   │   └── cd/
│   │       └── abcd
│   └── ht/
│       └── tp/
│           └── http
└── README.md                    # This document
```

## Dependencies

Dependencies are declared in each package's `prism-package.json`:

```json
{
    "name": "http-server",
    "version": "1.0.0",
    "repo": "https://github.com/user/http-server",
    "dependencies": ["http", "json", "net"]
}
```

When a package is published, its dependencies are included in the NDJSON entry.
When a package is installed, the CLI reads the registry index to discover transitive dependencies.
However, dependency resolution is not yet recursive — `prism install` installs one package at a time.

## FAQ

**Q: Why GitHub? Why not a traditional package server?**
A: GitHub provides free, reliable Git hosting with API access, Actions for automation, and pull requests as a built-in review queue. The entire registry runs on GitHub's infrastructure with zero maintenance cost.

**Q: Is a GitHub token required to install packages?**
A: No. `prism install` only needs `git` — the registry is a public Git repo. Tokens are only needed for publishing and bootstrapping.

**Q: What happens if two people publish the same version at the same time?**
A: GitHub serializes Actions on the same branch. The second PR will see the first PR's changes (via `fetch-depth: 0`) and append its entry after. No race conditions.

**Q: Can anyone publish any package name?**
A: Yes. The registry is open — like crates.io or npm, package names are first-come, first-served.

**Q: What if the GitHub Action fails?**
A: The PR stays open. The author sees the failure reason in the PR's Actions tab, fixes their `prism-package.json`, and re-runs `prism publish` (which opens a new PR).
