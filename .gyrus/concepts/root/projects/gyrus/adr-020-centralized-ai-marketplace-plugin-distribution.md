---
title: Centralized AI Marketplace Plugin Distribution
id: adr-020-centralized-ai-marketplace-plugin-distribution
type: adr
format: markdown
category: architecture
version: 4
status: accepted
last_updated: 2026-11-13T23:27:40Z
last_modified_by: ""
tenant: root
tags:
    - adr
    - architecture
    - plugins
    - marketplace
    - ai-marketplace
links:
    - concept: root/projects/gyrus/adr-008-native-client-plugin-marketplace-distribution
      rel: supersedes
    - concept: root/projects/gyrus/adr-004-agent-plugin-packaging-distribution-and-init-revamp
      rel: depends_on
---

# ADR 020: Centralized AI Marketplace Plugin Distribution

## Context

Under ADR 008, `armckinney/gyrus` hosted both the `gyrus` CLI/MCP runtime and its agent plugin distribution artifacts: the canonical plugin bundle under `packaging/plugins/gyrus`, per-harness marketplace catalogs (`.agents/`, `.claude-plugin/`, `.github/plugin/`), plugin/skill validation tests (`tests/plugins/`, `tests/skills/`, `tests/install/plugin_install_test.go`), and the `.goreleaser.yaml` `gyrus-plugin.tar.gz` release archive.

Modern agent harnesses install plugins directly from remote Git repositories, and multi-project agent plugins and skills have been consolidated into the centralized [`armckinney/ai-marketplace`](https://github.com/armckinney/ai-marketplace) repository. Retaining duplicate plugin manifests, harness catalogs, release tarballs, and harness-specific install instructions inside `armckinney/gyrus` couples unrelated release lifecycles and creates drift whenever harness conventions or marketplace layouts evolve.

## Decision

We deprecate and remove in-repo plugin packaging and harness marketplace distribution from `armckinney/gyrus` in favor of centralized distribution through `armckinney/ai-marketplace`:

1. **Centralized Marketplace Ownership**:
   - The canonical `gyrus` agent plugin (`plugins/gyrus` — skills, rules, and MCP configuration), harness marketplace manifests, and plugin validation live exclusively in [`armckinney/ai-marketplace`](https://github.com/armckinney/ai-marketplace).
2. **In-Repo Removal in `armckinney/gyrus`**:
   - Remove `packaging/plugins/`, `.agents/`, `.claude-plugin/`, `.github/plugin/`, `tests/plugins/`, `tests/skills/`, and `tests/install/plugin_install_test.go`.
   - Remove the `gyrus-plugin.tar.gz` archive hook and release artifact from `.goreleaser.yaml`, along with obsolete plugin/skill ignore rules in `.gitignore` and `.dockerignore`.
   - Keep CLI installation (`packaging/install.sh`, `tests/install/cli_install_test.go`, `tests/install/docker_mcp_test.go`) cleanly supported in `armckinney/gyrus`.
3. **Decoupled Cross-Repository Documentation**:
   - `armckinney/gyrus` links to [`armckinney/ai-marketplace`](https://github.com/armckinney/ai-marketplace) and its [`gyrus` plugin guide](https://github.com/armckinney/ai-marketplace/blob/main/docs/plugins/gyrus.md) rather than duplicating harness install commands.
   - `armckinney/ai-marketplace` links back to [`armckinney/gyrus#get-started`](https://github.com/armckinney/gyrus#get-started) for installing the prerequisite `gyrus` CLI binary.

## Consequences

### Positive
- **Separation of Concerns**: `armckinney/gyrus` focuses strictly on the CLI binary, Go SDK, MCP server, local web UI, and OKF/Git/PostgreSQL persistence backends, while `armckinney/ai-marketplace` owns multi-harness plugin packaging and catalogs.
- **Zero Cross-Repo Instruction Drift**: Marketplace or harness manifest changes in `armckinney/ai-marketplace` no longer require documentation or test updates in `armckinney/gyrus`.
- **Leaner Releases**: Eliminates redundant plugin tarball packaging in GoReleaser when harnesses install directly from the marketplace Git repository.

### Negative
- **Two-Step User Onboarding**: Users installing the plugin from `ai-marketplace` must separately install the `gyrus` CLI binary onto `PATH` via `armckinney/gyrus` so local stdio MCP servers can spawn `gyrus`.
