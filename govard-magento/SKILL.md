---
name: govard-magento
description: |
  Magento-specific Govard shortcuts and commands. Use when:
  - "Clear Magento cache", "flush redis cache"
  - "Run Magento CLI", "bin/magento commands"
  - "Deploy static content", "setup:di:compile"
  - "Reindex catalog", "indexer commands"
  - "Enable/disable modules"

  DEPENDENT on govard-toolbox for base commands.
compatibility: claude, codex, opencode, copilot
depends: [govard-toolbox]
metadata:
  audience: developers
  workflow: magento
---

# Govard Magento Commands

Magento-specific shortcuts and commands for Govard environments.

## Magento CLI

```bash
# Cache management
govard sh -c "bin/magento cache:flush"
govard sh -c "bin/magento cache:clean full_page"

# Specific cache types
govard sh -c "bin/magento cache:enable layout block_html"
govard sh -c "bin/magento cache:disable config"

# Module management
govard sh -c "bin/magento module:enable Vendor_Module"
govard sh -c "bin/magento module:disable Vendor_Module"
govard sh -c "bin/magento module:status"

# Setup commands
govard sh -c "bin/magento setup:di:compile"
govard sh -c "bin/magento setup:static-content:deploy -f"
govard sh -c "bin/magento setup:upgrade --keep-generated"

# Deploy mode
govard sh -c "bin/magento deploy:mode:set developer"
govard sh -c "bin/magento deploy:mode:set production"
govard sh -c "bin/magento deploy:mode:show"
```

## Code Generation

```bash
# Generate plugin
govard sh -c "bin/magento generate:plugin Vendor Module"

# Generate observer
govard sh -c "bin/magento generate:observer Vendor Module Event"

# Create admin grid
govard sh -c "bin/magento admin:user:create"
```

## Indexer Commands

```bash
# Check status
govard sh -c "bin/magento indexer:status"

# Reindex all
govard sh -c "bin/magento indexer:reindex"

# Single indexer
govard sh -c "bin/magento indexer:reindex catalog_product_price"

# Change mode
govard sh -c "bin/magento indexer:set-mode schedule"
govard sh -c "bin/magento indexer:set-mode realtime"
```

## Cron Commands

```bash
# Run cron manually
govard sh -c "bin/magento cron:run"
govard sh -c "bin/magento cron:run --group=default"

# Install crontab
govard sh -c "bin/magento cron:install"

# Remove crontab
govard sh -c "bin/magento cron:remove"
```

## Development Tools

```bash
# Template hints (dev only)
govard sh -c "bin/magento dev:template-hints:enable"
govard sh -c "bin/magento dev:template-hints:disable"

# Query logging
govard sh -c "bin/magento dev:query-log:enable"
govard sh -c "bin/magento dev:query-log:disable"

# JS/CSS bundling
govard sh -c "bin/magento dev:js:enable_js_bundling"
govard sh -c "bin/magento dev:css:minify_files"
```

## Database Operations

```bash
# Direct MySQL
govard db connect

# Run SQL
govard db query "SELECT * FROM m2_core_config_data WHERE path LIKE '%template%'"

# Import with streaming (fast)
govard db import --stream-db -e staging --drop

# Export from remote
govard db dump -e staging --no-noise --no-pii --local
```

## Configuration

```bash
# Show config
govard sh -c "bin/magento config:show system/smtp/host"

# Set config
govard sh -c "bin/magento config:set web/secure/base_url https://local.test/"
govard sh -c "bin/magento config:set design/theme/theme_id 0"

# Import/export config
govard sh -c "bin/magento app:config:dump"
govard sh -c "bin/magento app:config:import"
```

## Redis Cache

```bash
# Flush Redis cache only
govard redis flush

# Redis CLI
govard redis cli

# Check cache info
govard redis info
```

## Varnish (if configured)

```bash
# Purge all
govard varnish purge

# Purge by tag
govard sh -c "bin/magento cache:clean cache_tag_frontend"

# Varnish status
govard varnish status
```

## Logging

```bash
# View system log
govard sh -c "tail -f var/log/system.log"

# View exception log
govard sh -c "tail -f var/log/exception.log"

# View debug log
govard sh -c "tail -f var/log/debug.log"

# Custom module log
govard sh -c "tail -f var/log/my-module.log"
```

## Common Workflows

### After Pulling Code

```bash
govard sh -c "bin/magento setup:upgrade --keep-generated"
govard sh -c "bin/magento setup:static-content:deploy -f"
govard sh -c "bin/magento cache:flush"
```

### After Database Sync

```bash
govard config auto   # Rebuild env.php
govard sh -c "bin/magento cache:flush"
```

### Production Deployment Prep

```bash
govard sh -c "bin/magento maintenance:enable"
govard sh -c "bin/magento setup:upgrade"
govard sh -c "bin/magento setup:di:compile"
govard sh -c "bin/magento setup:static-content:deploy -f --theme=Vendor/Theme"
govard sh -c "bin/magento cache:flush"
govard sh -c "bin/magento maintenance:disable"
```