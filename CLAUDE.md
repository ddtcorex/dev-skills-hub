# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code plugin (`dev-skills-hub`) bundling 10 Magento 2 and Govard
development skills, distributed via a self-listing marketplace. There is no
build step, no test suite, and no application code — the repository *is* the
plugin, and its content is Markdown (`SKILL.md`) plus three JSON manifests and
one install script.

## Architecture

### Single source of truth: `skills/<name>/SKILL.md`

Every skill lives under `skills/<name>/SKILL.md` and **nowhere else**. This is
not a stylistic choice — it's a hard requirement of the Claude Code plugin
loader, which only auto-discovers `skills/<subdir>/SKILL.md` at the plugin
root. Earlier iterations of this repo kept skills in a flat top-level layout
(`magento2-dev-core/SKILL.md`) plus generated copies/symlinks in `skills/` for
plugin compatibility; that dual-location approach was deliberately abandoned
in favor of one location. Do not reintroduce a second copy of skill content
anywhere in the repo (no symlinks, no build-generated duplicates) — if a
change needs skill content in a different shape, change how it's *consumed*,
not where it lives.

### The plugin self-lists its own marketplace

`.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` live side
by side. `marketplace.json` has exactly one entry, with `"source": "./"`,
pointing back at the repo root where `plugin.json` itself lives — this repo is
simultaneously "a plugin" and "the marketplace that hosts that one plugin."
The marketplace's top-level `name` and the plugin's `name` are kept identical
(`dev-skills-hub`) on purpose, so `/plugin install dev-skills-hub@dev-skills-hub`
reads as one coherent thing rather than two unrelated names. The **version
field must be bumped in both files together** (`plugin.json`'s `version` and
`marketplace.json`'s `plugins[0].version` /  `metadata.version`) — nothing
enforces they match automatically.

### One SKILL.md format, four incompatible discovery paths

All 10 skills follow the [Agent Skills standard](https://agentskills.io) (a
`SKILL.md` file with `name`/`description` YAML frontmatter) — a format Claude
Code, OpenCode, Codex CLI, and GitHub Copilot all read identically. What
differs is which directory name each tool scans in a *consuming* project, and
none of them can be pointed at an arbitrary path:

| Tool | Project-level path |
|---|---|
| Claude Code | `.claude/skills/` |
| OpenCode | `.opencode/skills/` (also checks `.claude/skills/`, `.agents/skills/`) |
| Codex CLI | `.agents/skills/` only |
| GitHub Copilot | `.github/skills/` (also checks `.claude/skills/`, `.agents/skills/`, or `chat.agentSkillsLocations`) |

This repo's own bare `skills/` folder matches **none** of those — it only
works because Claude Code's *plugin* loader (a separate mechanism from
project-level skill scanning) specifically looks for `skills/` at plugin root.
`install.sh` exists to bridge this gap for direct (non-plugin) use: it never
tries to make one folder satisfy all four tools, it links per-tool into
whichever directory each one actually scans.

### Skill dependency chain

Some skills declare a non-enforced `depends:` frontmatter field documenting a
conceptual prerequisite (Claude Code's schema ignores it; it's a
cross-tool-sharing convention along with `compatibility` and `metadata`):

- `magento2-dev-core` is the foundation; `magento2-linter`,
  `magento2-performance-audit`, `magento2-security-scan`,
  `magento2-hyva-dev`, `magento2-frontend-dev`, and `magento2-backend-dev` all
  declare `depends: [magento2-dev-core]`.
- `govard-toolbox` is the foundation; `govard-magento` and `govard-laravel`
  both declare `depends: [govard-toolbox]`.

When editing a dependency's SKILL.md (`magento2-dev-core`, `govard-toolbox`),
check whether the change invalidates guidance in the skills that depend on it.

### `install.sh` design

One-line installer/updater (`curl -fsSL .../install.sh | bash`). Key points if
modifying it:

- **Clone-once-to-cache, link-out-per-tool**: clones this repo into
  `~/.dev-skills-hub` (override: `DEV_SKILLS_HUB_HOME`); re-running the same
  command is the update path (`git pull --ff-only` + re-link) — there is no
  separate `update.sh`.
- **TTY handling follows the rustup-init.sh pattern**: `[ -t 0 ]` for a real
  interactive stdin, else fall back to `< /dev/tty` if `[ -t 1 ]` (stdout is
  still a real terminal even though stdin was consumed by the `curl | bash`
  pipe), else silently use defaults (CI-safe). Don't replace this with a
  plain `read` — it will hang or misbehave under `curl | bash`.
- **Manifest-based safety**: every path it creates is recorded in
  `$CACHE_DIR/.manifest`. A pre-existing path not in that manifest is treated
  as user-owned and skipped unless `--force` — this is what stops the
  installer from clobbering a user's own hand-written skill of the same name.
  Don't remove this check to "simplify" the linking loop.
- Env vars mirror the flags and are prefixed `DEV_SKILLS_HUB_*` (`_HOME`,
  `_SCOPE`, `_TARGET`, `_SKILLS`, `_MODE`) — keep this prefix if adding new
  configurable behavior.

## Commands

There is no build/lint/test framework — validation is structural (does the
plugin manifest resolve correctly?) and, for `install.sh`, behavioral (does it
actually link/unlink files correctly?).

```bash
# Validate plugin + marketplace manifest (must be run from repo root)
claude plugin validate . --strict

# Inspect what the plugin loader actually resolves (skills found, token cost)
claude --plugin-dir . plugin details dev-skills-hub

# Full local install/uninstall round-trip against the working tree (not a
# published release) -- always clean up after testing, this registers real
# state in the local Claude Code config:
claude plugin marketplace add .
claude plugin install dev-skills-hub@dev-skills-hub
# ... test ...
claude plugin uninstall dev-skills-hub@dev-skills-hub
claude plugin marketplace remove dev-skills-hub

# install.sh: syntax check and dry test in an isolated scratch dir (never
# test --scope personal against your real $HOME -- override HOME and
# DEV_SKILLS_HUB_HOME to a scratch path first). Note: install.sh's REPO_URL
# points at GitHub, so testing local/uncommitted changes to skills/ requires
# either a temporary local git remote (init+commit a scratch copy and point
# REPO_URL at it) or pushing first -- git clone never sees uncommitted work.
bash -n install.sh
bash install.sh --help
```

## Release checklist

Pushing a `vX.Y.Z` tag triggers `.github/workflows/release.yml`, which
validates the manifests, extracts the matching `## [X.Y.Z]` section from
`CHANGELOG.md`, and publishes a GitHub Release automatically — there is no
manual release step in the GitHub UI.

1. Add a new `## [X.Y.Z] - YYYY-MM-DD` section at the top of `CHANGELOG.md`
   (Keep a Changelog format: `### Added` / `### Changed` / `### Fixed` etc.).
2. Bump `"version"` in `.claude-plugin/plugin.json`.
3. Bump `"version"` in `.claude-plugin/marketplace.json` — both
   `plugins[0].version` and top-level `metadata.version` — to the same value.
4. If `skills/` changed, run `claude plugin validate . --strict` and
   `claude --plugin-dir . plugin details dev-skills-hub` locally first.
5. Commit, then tag and push:
   ```bash
   git tag -a vX.Y.Z -m "vX.Y.Z - <one-line summary>"
   git push origin master
   git push origin vX.Y.Z
   ```
6. Confirm the workflow succeeded and the release published:
   ```bash
   gh run list --repo ddtcorex/dev-skills-hub --limit 1
   gh release view vX.Y.Z --repo ddtcorex/dev-skills-hub
   ```
7. If the workflow fails on the changelog-extraction step, it's almost always
   because the `## [X.Y.Z]` header in `CHANGELOG.md` doesn't exactly match the
   pushed tag's version (the workflow strips a leading `v` from the tag and
   looks for `[X.Y.Z]` literally).
