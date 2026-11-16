---
type: Concept
title: Area Map
x-ai-rulez:
  kind: context
  id: area-map
  metadata:
    priority: high
---

# Area Map

- `repos.txt` — the repos this workspace clones, one `<dir> <remote>` per line; all are under `github.com/polylint`
- `mise.toml` — settings only, no `[tools]` (inherited by every subrepo); `mise-tasks/` — root-only tasks that pin their own tools: `setup:*`,
  `repos:*` (`status`, `pull`), `worktree:*`, `ai-rulez:*` (`check`, `update`), `governance:*`, `lint`/`format`; product
  development tasks live in each repository
- `toolchains.toml` — the `xberg-io/toolchains` baseline the root task pins follow
- `.ai-rulez/` — governance for this root only; shared modules come from `xberg-io/agent-conventions`
- `poly.toml` — root lint, format and hook config; subrepos are excluded from discovery
- `.github/workflows/` — CI for this repo's governance sources
- `.worktrees/<repo>/<name>` — every git worktree (gitignored); layout and naming: the `polyrepo-structure` context

Repositories:

- Tools: `polylint` (the `poly` linter, formatter, hooks and MCP server), `skif` (JS/TS dependency linter),
  `adrlint` (MADR decision-record linter), `uncomment` (tree-sitter comment removal); polylint embeds the other three
  as engines
- Distribution: `homebrew-tap` (`polylint/tap`), `scoop-bucket`; release workflows push to both through the org
  GitHub App (`BOT_APP_ID` / `BOT_APP_PRIVATE_KEY`)
- Org profile: `dot-github` (the `polylint/.github` repository: `profile/README.md`, org-default community files,
  brand assets)
- Shared CI actions, agent-conventions and toolchains stay in the xberg-io organization and are consumed by tag
