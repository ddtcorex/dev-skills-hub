---
name: govard-toolbox
description: |
  High-level shortcuts and references for Govard development environment orchestrator. Use when:
  - "Start/stop environment", "govard up", "govard down"
  - "Run commands in container", "govard sh"
  - "Database operations", "db dump", "db import"
  - "Sync with remote", "bootstrap from staging"
  - "Debug configuration", "Xdebug setup"

  This is the BASE skill - for framework-specific shortcuts, also load govard-magento or govard-laravel.
compatibility: claude, codex, opencode, copilot
metadata:
  audience: developers
  workflow: general
---

# Govard Toolbox

Govard is a containerized development environment orchestrator. This skill provides high-level shortcuts and references.

## Quick Reference

### Environment Lifecycle

| Shortcut | Full Command | Purpose |
|----------|--------------|---------|
| `govard up` | `govard env up` | Start project |
| `govard down` | `govard env down` | Stop project |
| `govard sh` | `govard shell` | Interactive shell |
| `govard ps` | `govard env ps` | List containers |

### Common Commands

```bash
# Start environment
govard up

# Stop environment
govard down

# Stop with volumes (clean slate)
govard down -v

# Shell into PHP container
govard sh

# Run single command
govard sh -c "ls -la"

# View logs
govard logs -f

# Restart services
govard restart
```

### Database Operations

```bash
# Connect to MySQL
govard db connect

# Run query
govard db query "SELECT * FROM admin_user LIMIT 1"

# Import dump
govard db import --file backup.sql --drop

# Export database
govard db dump --no-noise -e staging

# Direct sync from remote
govard db import --stream-db -e staging --drop
```

### Remote Sync

```bash
# Add remote
govard remote add staging ssh://user@staging.server/path

# Test connection
govard remote test staging

# Sync everything
govard sync --source staging --full

# Plan before sync (preview)
govard sync --source staging --full --plan

# Skip noise (cache, logs)
govard sync --source staging --full --no-noise --no-pii
```

### Bootstrap

```bash
# From staging (full setup)
govard bootstrap --clone -e staging --no-pii --no-noise --yes

# Preview plan
govard bootstrap --clone -e staging --plan
```

## Tool Execution

```bash
# Run framework CLI
govard tool magento cache:flush
govard tool magerun cache:clear
govard tool artisan migrate
govard tool drush cr

# Node tools
govard tool npm install
govard tool composer install
```

## Services

```bash
# Redis
govard redis flush
govard redis cli

# Varnish
govard varnish purge

# Open URLs
govard open app      # Main site
govard open admin    # Admin panel
govard open db       # PHPMyAdmin
govard open mail     # Mailhog
```

## Debugging

```bash
# Xdebug control
govard debug on
govard debug off
govard debug status

# Diagnostics
govard diag
govard diag --fix    # Auto-fix issues
```

## Configuration

```bash
# Auto-config after DB sync
govard config auto

# Show config
govard config get system/version

# Interactive config
govard config set domain local.test
```

## Detailed References

See bundled documents:
- [COMMANDS.md](COMMANDS.md) - Exhaustive command reference
- [GUIDES.md](GUIDES.md) - Case studies and patterns
- [FAQ.md](FAQ.md) - Troubleshooting

For Magento-specific: Load `govard-magento` skill
For Laravel-specific: Load `govard-laravel` skill