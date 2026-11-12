# wiki
Gyrus Wiki

## Gyrus project knowledge

The Gyrus project's documentation lives in [`.gyrus/concepts/root/projects/gyrus`](.gyrus/concepts/root/projects/gyrus). All 67 migrated concepts use project scope (`root/projects/gyrus/<id>`); there are no shared reference concepts in this collection. Schemas remain collection-wide under [`.gyrus/schemas`](.gyrus/schemas).

Start with the [architecture overview](.gyrus/concepts/root/projects/gyrus/spec-000-overview.md), [setup guide](.gyrus/concepts/root/projects/gyrus/guide-002-setup-onboarding.md), or [Git initiative guide](.gyrus/concepts/root/projects/gyrus/guide-006-git-initiatives.md).

Use this configuration in a Gyrus client checkout:

```yaml
tenant: root
project: gyrus
storage:
  provider: git
git:
  repo_url: https://github.com/armckinney/wiki.git
  trunk_branch: main
  commit_strategy: direct
index:
  provider: okf
graph:
  provider: okf
search:
  provider: okf
```

Writes commit directly to `main`. Git credentials come from the configured credential helper; no secrets belong in configuration. To work on a reviewed initiative instead, supply `--git-branch initiative/name` on each CLI command or select a branch for an MCP session. PR creation and merging remain manual.
