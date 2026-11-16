<!-- markdownlint-disable MD033 MD041 -->
<div align="center">

<img src="https://raw.githubusercontent.com/polylint/.github/main/profile/assets/banner.svg" alt="polylint" width="820">

</div>

# polylint polyrepo

Polyrepo workspace for the [polylint](https://github.com/polylint) GitHub organization: high-performance Rust
linting and formatting tools built for agentic workflows. This repo tracks only workspace orchestration: the
repository registry, root AI governance, setup tasks and lint config. Every subdirectory below is an independent git
repository, cloned by `mise run setup:clone` and ignored here.

The point is one checkout to launch coding agents from: each agent starts at this root, reads the workspace
conventions, and works in its own worktree of whichever repository the task is about.

The layout and tasks mirror the [xberg-io polyrepo](https://github.com/xberg-io/polyrepo). The repositories here
share that organization's tooling: tool versions follow an [xberg-io/toolchains](https://github.com/xberg-io/toolchains)
baseline, CI uses [xberg-io/actions](https://github.com/xberg-io/actions), and AI governance includes
[xberg-io/agent-conventions](https://github.com/xberg-io/agent-conventions). Those stay in the xberg-io organization
and are not cloned here.

## Repositories

[`repos.txt`](repos.txt) is the source of truth. Each repository is cloned into a directory of the same name, except
`polylint/.github` (the organization profile), cloned as `dot-github/`.

### Tools

| Repository | Description |
|------------|-------------|
| [polylint](https://github.com/polylint/polylint) | One linter and formatter for every language (the `poly` command): one binary and one `poly.toml`, engines downloaded on demand, git hooks and an MCP server |
| [skif](https://github.com/polylint/skif) | Dependency and package-manager linter, auditor and enforcer for JavaScript and TypeScript monorepos |
| [adrlint](https://github.com/polylint/adrlint) | Linter and formatter for MADR architecture decision records |
| [uncomment](https://github.com/polylint/uncomment) | Removes and lints comments in 300+ languages using tree-sitter grammars |

polylint embeds skif, adrlint and uncomment as engines; each tool also ships on its own.

### Distribution and profile

| Repository | Description |
|------------|-------------|
| [homebrew-tap](https://github.com/polylint/homebrew-tap) | Homebrew formulae: `brew install polylint/tap/<tool>` |
| [scoop-bucket](https://github.com/polylint/scoop-bucket) | Scoop manifests (Windows) |
| [.github](https://github.com/polylint/.github) | Organization profile README, brand assets and default community files (`dot-github/`) |

Release workflows push formulae and manifests through the organization GitHub App (`BOT_APP_ID` /
`BOT_APP_PRIVATE_KEY` org secrets) with `xberg-io/actions/publish-homebrew-source-formulas` and
`publish-scoop-manifests`.

To add a repository, append `<dir> <remote>` to `repos.txt`, add `<dir>/**` to `[discovery] exclude` in
`poly.toml`, list it above and in `.ai-rulez/context/area-map.md`, then run `mise run setup:clone`.

## Setup

Prerequisites:

- [mise](https://mise.jdx.dev). Each task in `mise-tasks/` pins the tools it runs with (Node, Python,
  [polylint](https://github.com/polylint/polylint), shfmt, ShellCheck, jq) at the toolchains baseline named in
  `toolchains.toml`, and mise installs them on first use. `mise.toml` has no `[tools]`, because every subrepository
  inherits it, and the root `mise.lock` is gitignored: every subrepository install rewrites a parent lockfile to its
  own platforms.
- SSH access to `github.com/polylint` (every repository here is public; SSH is the remote form in `repos.txt`).
- `ai-rulez` on `PATH` for the git hooks: `npm install -g ai-rulez@latest` (Node 22 or later). The tasks themselves
  always run ai-rulez at npm latest through `npx -y ai-rulez@latest`.

```sh
git clone git@github.com:polylint/polyrepo.git polylint
cd polylint
mise trust
mise run setup
```

`mise run setup` runs four steps, in order, and runs every step even when an earlier one fails, so a single
inaccessible repository or broken subrepository setup never leaves the rest of the workspace unprovisioned. It exits
non-zero and names the failed steps at the end; each step reports the repositories it failed in.

1. `setup:clone` clones every repository in `repos.txt` that is not checked out yet.
2. `setup:hooks` installs Poly git hooks in the root and in every repository whose `poly.toml` declares `[hooks]`.
   Poly reads `[hook_preferences]` only from the repository it installs into, so for each repository with external
   hook sources the task records `channels = ["system"]` in that repository's gitignored `poly.local.toml` first. It
   refuses to write one that the repository does not gitignore.
3. `setup:subrepos` runs `mise run setup` in each repository that defines its own `setup` task.
4. `setup:ai-rulez` runs `npx -y ai-rulez@latest generate --no-local --gitignore` in the root, then
   `generate --recursive --no-local --gitignore` inside each repository with an `.ai-rulez/config.toml`. It skips
   `plugin/` (built by each product's plugin pipeline) and ai-rulez's `tests/fixtures/` and
   `internal/migrate/testdata/` (test configs a plain generate would rewrite). `--gitignore` keeps each repository's
   managed `.gitignore` block in step with its Claude, Codex and OpenCode outputs.

Rerun any step on its own; every one is idempotent. Afterwards, `mise run ai-rulez:check` and `mise run repos:status`
confirm the result.

## Tasks

Tasks are scripts under `mise-tasks/`; `mise tasks` lists them. They run against the polyrepo root, the parent of
this repository's git directory, also when invoked from a worktree of it; `repos.txt` is read from the checkout that
runs them. Subrepositories inherit the root `mise.toml`, so each task refuses to run when invoked from another
repository or below a nearer mise config: a subrepository defines its own `setup`, `lint` and the like.

| Task | Does |
|------|------|
| `setup` | `setup:clone`, `setup:hooks`, `setup:subrepos`, `setup:ai-rulez`; reports every failed step |
| `setup:clone` | Clone every registry repository that is not checked out yet; reports the ones that fail |
| `setup:hooks` | Provision hook preferences and install Poly hooks here and in every repository that declares hooks |
| `setup:subrepos` | Run `mise run setup` in each repository that defines a mise `setup` task; reports the ones that fail |
| `setup:ai-rulez` | `ai-rulez generate --gitignore`: the root, then `--recursive` inside each repository |
| `ai-rulez:check` | Fail when any root's generated output is stale, or when a generated file is neither tracked nor gitignored |
| `ai-rulez:update` | Move every repository's version-range pins (agent-conventions includes; skills that use ranges) to the newest allowed release, then run `setup:ai-rulez` |
| `repos:status` | Per repository: branch, changed files, ahead/behind its upstream (no fetch) and worktree count; flags missing repositories and any the root `.gitignore` does not ignore |
| `repos:pull` | Fetch all repositories and fast-forward those that are only behind; reports diverged or blocked ones |
| `worktree:add <repo> <branch> [--base <ref>]` | Create a worktree at `.worktrees/<repo>/<branch>`, with characters outside `A-Za-z0-9_-` replaced by `-`; a new branch starts from `origin/HEAD` |
| `worktree:remove <repo> <branch>` | Remove that worktree (the branch stays) |
| `governance:hooks:install` | Provision hook preferences and install Poly hooks in the root only (`setup:hooks` runs it first) |
| `lint`, `format`, `format:check` | Poly over the root files, with shfmt and ShellCheck for the task scripts |

Development tasks for a project live in that repository: `mise -C <repo> tasks`.

### Checking generated configuration

`mise run ai-rulez:check` prints one line per root:

```text
.                            63 outputs: 60 ignored, 3 tracked
skif                         83 outputs: 79 ignored, 4 tracked
uncomment                    83 outputs: 78 ignored, 4 tracked, 1 NEITHER: AGENTS.md
```

Tracked outputs are committed on purpose: the `.ai-rulez/**/index.md` files, and whatever a repository lists in
`gitignore_exclude` (some repositories commit their `AGENTS.md`/`CLAUDE.md` or agent and skill files). An output that is NEITHER would show up as a new untracked file after generate: fix that repository's
`.gitignore` or `gitignore_exclude`, then rerun `mise run setup:ai-rulez`.

## Working with agents

Start an agent (Claude Code, Codex or OpenCode) at this root. The generated `AGENTS.md`/`CLAUDE.md` carry the
workspace conventions, and the ai-rulez MCP server serves the shared skills on demand. A typical session:

```sh
mise run repos:pull
mise run repos:status
mise run worktree:add skif fix/catalog-drift
claude   # or codex, opencode; point it at .worktrees/skif/fix-catalog-drift
```

Run one worktree per agent so parallel agents never share a checkout, and remove it with `mise run worktree:remove`
once the branch is merged. Inside a repository, that repository's own `AGENTS.md`/`CLAUDE.md` and mise tasks take
precedence. The `polyrepo-development` rule in `.ai-rulez/rules/` holds the workspace conventions.

## AI governance

Authored configuration is in [`.ai-rulez/`](.ai-rulez/); `AGENTS.md`, `CLAUDE.md` and the per-harness files
(`.claude/`, `.codex/`, `.opencode/`, `.agents/`, `.mcp.json`, `opencode.json`) are generated and gitignored. The
root includes `modules/core` from agent-conventions by version range (`version = "^1"`), and the lock records the
exact commit. It follows the same layout as the xberg-io root: `compact = true`, glob-scoped rules served through
`[rules] scoped_fallback = "skill"`, the ai-rulez MCP server restricted to `find_skill` and `load_skill`, and an
area map in `.ai-rulez/context/`.

To change a shared rule, edit it in agent-conventions, push, tag a release, then run `mise run ai-rulez:update`:
it runs `ai-rulez update` in the root and every repository, then regenerates through `setup:ai-rulez`. Preview with
`ai-rulez lock --outdated` or `ai-rulez update --dry-run`. To regenerate and verify here:

```sh
ai-rulez validate --strict --no-local
ai-rulez lock
ai-rulez generate --no-local --strict-config
ai-rulez generate --check --no-local --strict-config
```

CI (`.github/workflows/ai-rulez-validation.yml`) validates the sources and generated outputs with the shared
`setup-ai-rulez` and `validate-ai-rulez` actions.
