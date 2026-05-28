# Prism Package Manager — Architecture & Specification

## Architecture Overview

The Prism Package Manager is completely serverless—eliminating the need for custom backends, databases, or API gateways. The entire registry state, lifecycle, and automation workflow runs directly through GitHub infrastructure.

```text
[Your Machine]                  [GitHub Registry Repo]           [Package Source Repo]
      │                                    │                               │
      ├─ prism publish ───────────────────►│─ Fork → Branch → PR ────────► GitHub Action
      │                                    │  Auto-merges & builds tree    │
      │                                    │                               │
      └─ prism install ───────────────────►│─ Pull registry ─────────────► git clone
         Writes to: ~/.prism/registry/     │  Resolves path dynamically    │  Writes to: ~/.local/lib/prism
Registry Structure
To avoid file system performance issues associated with flat directories, packages are organized inside a public GitHub repository (prism-package-repository) using a Cargo-style path partitioning strategy based on name length:
```

Plaintext
index/
├── 1/a             # 1-character names (e.g., "a")
├── 2/go            # 2-character names (e.g., "go")
├── 3/h/hub         # 3-character names, grouped by the first letter
└── ht/tp/http      # 4+ character names, sharded by first two / next two letters
Metadata Format
Each package file consists of Newline-Delimited JSON (NDJSON). Every line represents a unique published version:

```JSON
{"vers":"0.1.0","repo":"[https://github.com/prism-packages/http](https://github.com/prism-packages/http)","deps":["json"]}
{"vers":"0.2.0","repo":"[https://github.com/prism-packages/http](https://github.com/prism-packages/http)","deps":["json","ssl"]}
Note: The final line of the file is always interpreted as the latest stable release.
```

Commands Specification
prism publish
Publishes the package in the current working directory to the global registry.

Bash
prism publish
Lifecycle Steps:
Validation: Parses and validates prism-package.json to ensure required fields (name, version, repo) contain only shell-safe characters.

Forking: Forks the central registry repository via the GitHub API using the developer's local PRISM_GITHUB_TOKEN.

Branching: Creates a new isolated branch named publish/<name>-<version> containing the metadata change.

Pull Request: Opens a PR against the upstream registry. A GitHub Action automatically validates the request, computes the deterministic destination path, appends the entry, and squash-merges the PR into main.

prism install
Installs a remote package from the global registry, or links a local package folder.

Bash
prism install <package_name>
prism install <local_path>
Example:
Bash
prism install http
Lifecycle Steps for Remote Packages:
Registry Sync: Synchronizes the central registry state to ~/.prism/registry/ (git clone on first invocation; git pull on subsequent runs).

Path Resolution: Evaluates the package name length to locate the precise target file path (e.g., "http" resolves directly to index/ht/tp/http).

Metadata Parsing: Reads the final line of the resolved file to extract the latest version string and source repository URL.

Cloning: Executes a secure git clone of the target package source into ~/.local/lib/prism/packages/http/.

Lifecycle Steps for Local Packages:
Bash
prism install ./my-package
Copies the specified directory directly into the local project cache without hitting the remote network.

Security Architecture
Injection Mitigation: Package names, version strings, and repository URLs are strictly validated against a comprehensive alphanumeric allowlist before interacting with any low-level system() operations.

Process Listing Protection: Git authentication processes utilize GIT_ASKPASS combined with transient background scripts, ensuring sensitive authentication tokens are never exposed in plaintext via standard process monitoring tools (like ps).

Payload Isolation: Outgoing GitHub API HTTP interactions utilize strict single-quote parameter escaping to prevent accidental or malicious shell breakouts during token execution.

Cost & Infrastructure Scale
The entire ecosystem leverages GitHub’s core infrastructure tiers to maintain zero operational runtime costs:

Registry State Storage: Maintained inside a public, git-backed repository.

Publishing Automation: Driven entirely by on-demand GitHub Actions workflows configured in .github/workflows/publish.yml.

Package Hosting: Distributed across authors' individual code repositories, linked globally via simple URL tracking.
