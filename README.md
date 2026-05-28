# Prism Package Manager — How It Works

---

## Architecture

The package manager is serverless — no backend, no database, no API gateway. Everything runs through GitHub:

```
[Your Machine]          [GitHub Registry Repo]         [Package Source Repo]
     │                         │                              │
     ├─ prism publish ────────►│  Fork → Branch → PR ──►  GitHub Action
     │                         │  auto-merges, builds tree   │
     │                         │                              │
     ├─ prism install ────────►│  Pull registry ─────────►  git clone
     │   ~/.prism/registry/    │  Resolve path               │  → ~/.local/lib/prism
```

---

## Registry Structure (Cargo-style path partitioning)

Packages live in a GitHub repo (`prism-package-repository`) organized by name length to avoid flat-directory performance issues:

```
index/
├── 1/a            ← 1-char names (e.g., "a")
├── 2/go           ← 2-char names (e.g., "go")
├── 3/h/hub        ← 3-char names, grouped by first letter
└── ht/tp/http     ← 4+ char names, grouped by first two / next two letters
```

Each file is newline-delimited JSON (like Cargo), one version per line:

```json
{"vers":"0.1.0","repo":"https://github.com/prism-packages/http","deps":["json"]}
{"vers":"0.2.0","repo":"https://github.com/prism-packages/http","deps":["json","ssl"]}
```

> The last line is always the latest version.

---

## `prism publish`

```
prism publish
```

1. Validates `prism-package.json` (name, version, repo — all checked for shell-safe characters)
2. Forks the registry repo via GitHub API using your `PRISM_GITHUB_TOKEN`
3. Creates a branch `publish/<name>-<version>` with the metadata
4. Opens a PR — a GitHub Action auto-validates, computes the correct path, appends the entry, and squash-merges

---

## `prism install`

```
prism install http
```

1. Syncs registry to `~/.prism/registry/` (git clone on first use, git pull after)
2. Resolves path — knows exactly where to look: `index/ht/tp/http` for "http"
3. Parses the last line to get the latest version + repo URL
4. Clones the package repo into `~/.local/lib/prism/packages/http/`

For local packages: `prism install ./my-package` just copies the folder.

---

## Security

- **No shell injection:** package names, versions, and repo URLs are validated against strict character allowlists before ever touching `system()`
- **No token in process listings:** Git auth uses `GIT_ASKPASS` with a temp script instead of embedding the token in clone URLs
- **Curl payloads:** single-quote escaping prevents shell break-out in GitHub API calls

---

## No Infrastructure Costs

The entire system runs on GitHub's free tier:

- **Registry storage** = a public GitHub repo
- **Publishing automation** = a GitHub Action in `.github/workflows/publish.yml`
- **Package hosting** = package authors' own repos (linked by URL in the registry)
