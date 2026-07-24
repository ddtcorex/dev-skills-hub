# Changelog

All notable changes to this project are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[Semantic Versioning](https://semver.org/).

## [0.1.1] - 2026-07-24

### Added
- `CLAUDE.md` documenting the repo's architecture, plugin/marketplace structure,
  `install.sh` design, and release checklist for future contributors.
- `.gitignore` for common local/editor artifacts.

### Changed
- Strengthened cross-skill dependency signaling: all 8 skills declaring
  `depends:` (`govard-laravel`, `govard-magento`, `magento2-backend-dev`,
  `magento2-frontend-dev`, `magento2-hyva-dev`, `magento2-linter`,
  `magento2-performance-audit`, `magento2-security-scan`) now state their
  prerequisite as an explicit `REQUIRED BACKGROUND` instruction in the skill
  body, since Claude Code's schema doesn't act on `depends:` frontmatter alone.
- Removed `Usage`/"Trigger phrases" sections that duplicated each skill's
  frontmatter `description` verbatim; kept the one genuine step-by-step
  workflow (renamed to `Workflow`) in `magento2-performance-audit`.

## [0.1.0] - 2026-07-14

### Added
- Packaged the skill collection as a Claude Code plugin (`.claude-plugin/plugin.json`,
  self-listing `marketplace.json` with `"source": "./"`), installable with
  `/plugin marketplace add ddtcorex/dev-skills-hub` + `/plugin install dev-skills-hub@dev-skills-hub`.
- `install.sh`, a one-line installer/updater (`curl -fsSL .../install.sh | bash`):
  clones into `~/.dev-skills-hub` and links each skill into whichever directory
  Claude Code, OpenCode, Codex CLI, or GitHub Copilot scans, with project/personal
  scope, symlink/copy modes, selective skill/target install, and `--uninstall`.
- 10 skills: `magento2-dev-core`, `magento2-linter`, `magento2-performance-audit`,
  `magento2-security-scan`, `magento2-hyva-dev`, `magento2-frontend-dev`,
  `magento2-backend-dev`, `govard-toolbox`, `govard-magento`, `govard-laravel`.

### Changed
- Restructured from a flat top-level layout to `skills/<name>/SKILL.md` — the
  layout Claude Code's plugin loader requires and the one the wider
  [Agent Skills standard](https://agentskills.io) (also adopted by OpenCode,
  Codex CLI, and GitHub Copilot) expects.
- Renamed the project — repo, plugin, and marketplace — from `ai-skills` to
  `dev-skills-hub`, and repositioned the description around "development
  skills" generally rather than only the Magento 2 / Govard skills bundled
  today.
