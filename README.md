# AI Skills Hub

A centralized, production-grade flat repository designed to host and organize custom AI Skills for multiple AI Agents (Claude Code, OpenCode, VS Code Codex, GitHub Copilot).

This repository is built to be **fully extensible**, allowing you to continuously add new tech stacks, tools, and domain-specific skills over time.

---

## 📂 Flat Directory Structure

All skill directories are located at the root level of the repository for direct auto-discovery by AI Agents.

```
ai-skills/
├── README.md                        # This file (Global documentation and guide)
│
├── 📦 CORE STANDARDS & ARCHITECTURES (Foundation patterns)
│   └── magento2-dev-core/           # Magento 2 core guidelines (DI, Repositories, Security)
│
├── 🛠️ QUALITY ASSURANCE & AUDITING (Linting and static analysis)
│   ├── magento2-linter/             # PHPCS, PHPStan static checking rules
│   ├── magento2-performance-audit/  # Web vitals, infrastructure and DB profiling
│   └── magento2-security-scan/      # Static vulnerability code scanning
│
├── 🎨 FRONTEND & BACKEND FRAMEWORKS (Domain-specific code)
│   ├── magento2-hyva-dev/           # Alpine.js, Tailwind CSS, CSP payment pages
│   ├── magento2-frontend-dev/       # Luma Knockout.js, LESS, RequireJS
│   └── magento2-backend-dev/        # REST, GraphQL resolvers, Cron, Queues
│
└── 🔧 DEV ENVIRONMENT & CLI TOOLS (Toolchain orchestration)
    ├── govard-toolbox/              # Base container orchestrator toolbox
    ├── govard-magento/              # Magento-specific dev env commands
    └── govard-laravel/              # Laravel-specific dev env commands
```

---

## ⚡ Extension Guide: Adding New Skills

You can easily expand this hub with any new programming language, framework, or utility domain (e.g., React, Python, Docker, CI/CD, AWS, etc.).

### Step 1: Create a Flat Directory
Create a folder directly at the root of the project using kebab-case:
`[domain]-[tool/stack]-[purpose]` (e.g., `react-nextjs-tailwind`, `docker-ops-deployment`).

### Step 2: Create a `SKILL.md` File
Every skill folder **must** contain a `SKILL.md` file at its root with valid YAML frontmatter:

```markdown
---
name: your-skill-name
description: |
  Describe exactly when the AI Agent should trigger this skill.
  Make it descriptive and comprehensive so agents easily understand context.
compatibility: claude, codex, opencode, copilot
depends: [any-dependencies-if-applicable]
metadata:
  audience: developers
  workflow: your-workflow
---

# Your Skill Title

## Capabilities
Describe what this skill enables the agent to do.

## Best Practice Patterns
Provide code templates, standards, and typical commands.

## Verification
Step-by-step commands to verify output works properly.
```

### Step 3: Add to `README.md`
Add your new skill to the directories list above and update the compatibility matrix below.

---

## 🤖 Agent Compatibility Matrix

| Skill | Claude Code | VS Code Codex | OpenCode | GitHub Copilot |
|-------|-------------|---------------|----------|----------------|
| **Core & Standards** | | | | |
| [magento2-dev-core](magento2-dev-core/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| **Linting & Auditing** | | | | |
| [magento2-linter](magento2-linter/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| [magento2-performance-audit](magento2-performance-audit/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| [magento2-security-scan](magento2-security-scan/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| **Frameworks** | | | | |
| [magento2-hyva-dev](magento2-hyva-dev/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| [magento2-frontend-dev](magento2-frontend-dev/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| [magento2-backend-dev](magento2-backend-dev/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| **Toolchains** | | | | |
| [govard-toolbox](govard-toolbox/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| [govard-magento](govard-magento/SKILL.md) | ✅ | ✅ | ✅ | ✅ |
| [govard-laravel](govard-laravel/SKILL.md) | ✅ | ✅ | ✅ | ✅ |

---

## 💾 Installation & Auto-Discovery

AI Agents naturally scan the root level of folders in your workspace. Since this repository follows a **strict flat structure**, pointing an agent to this root workspace is sufficient for them to index and auto-discover all custom capabilities.

### Global Installation (Claude Code)
To register a skill globally so it triggers in any directory:
```bash
cp -r [skill-folder] ~/.claude/skills/
```

---

*Let's continuously expand this hub to automate more of our engineering workflows!* 🚀