# Exhaustive Govard Command Reference

Full canonical reference for all Govard subcommands.

## 1. Environment Lifecycle (`govard env`)

| Command | Purpose | Flag | Effect |
| :--- | :--- | :--- | :--- |
| `up` | Start project | `--pull` | Pull image before startup |
| `down` | Remove project | `-v, --volumes` | Remove database volumes |
| `ps` | List status | `--all` | Show all, including stopped |
| `logs` | View output | `-f, --tail` | Follow logs in real-time |
| `exec` | Run command | `-u, --user` | Run as specific user |
| `run` | One-off run | `--name` | Custom container name |
| `restart` | Restart all | - | - |
| `shell` | App bash | `--no-tty` | Run without generic TTY |
| `start`/`stop` | Services | - | - |
| `build` | Build context | `--no-cache` | Build images from scratch |
| `cp` | Copy file | - | `govard env cp src dest` |
| `cleanup` | Cleanup env | `--orphans` | Remove orphaned resources |

## 2. Database utilities (`govard db`)

| Command | Purpose | Example |
| :--- | :--- | :--- |
| `connect` | MySQL Shell | `govard db connect` |
| `query` | Run SQL | `govard db query "SELECT ..."` |
| `import` | Load SQL | `govard db import --file backup.sql --drop` |
| `dump` | Save SQL | `govard db dump --no-noise -e staging` |
| `info` | Connection Info | `govard db info` |
| `top` | Live Queries | `govard db top -e staging` |
| `import --stream-db` | Direct sync | `govard db import --stream-db -e staging --drop` |

## 3. Tool Execution (`govard tool`)

- **`magento`** / **`magerun`**: Magento CLI / OpenMage
- **`artisan`**: Laravel
- **`drush`**: Drupal
- **`symfony`**: Symfony
- **`wp`**: WordPress
- **`composer`**: PHP Package Manager
- **`npm`** / **`npx`** / **`yarn`** / **`pnpm`**: Node.js Package Managers

## 4. Synchronization & Remotes (`govard sync` / `govard remote`)

| Remote Command | Purpose | Sync Flag | Purpose |
| :--- | :--- | :--- | :--- |
| `remote add` | Add staging | `-s, --source` | Source env (e.g. `staging`) |
| `remote test` | Test auth | `-d, --destination` | Destination env (default: `local`) |
| `remote list` | Check remotes | `--db`, `--media` | Scope of data sync |
| `remote exec` | Run on remote | `--full` | Sync code + db + media |
| `remote audit` | Check ops log | `--plan` | Preview before execution |
| `remote copy-id` | Copy SSH key | `--no-noise` | Skip cache, logs, tags |
| - | - | `--no-pii` | Skip customer/order PII |

## 5. Snapshots (`govard snapshot`)

- `snapshot create`: Capture local state or remote `-e <env>`
- `snapshot list`: List available snapshots
- `snapshot restore <name>`: Roll back current environment
- `snapshot pull <name> -e <env>`: Fetch remote snapshot to local
- `snapshot push <name> -e <env>`: Send local snapshot to remote

## 6. Services & Shortcuts

- **`govard svc`**: `up`, `restart`, `logs`, `sleep`, `wake`
- **`govard redis`**: `flush`, `cli`, `info`
- **`govard varnish`**: `purge`, `status`
- **`govard open`**: `app`, `admin`, `mail`, `db`, `db --pma`
- **`govard debug`**: `on`, `off`, `status`, `shell`
- **`govard doctor`**: `trust` (Root CA), `--fix`, `--pack`
- **`govard config`**: `get`, `set`, `profile`, `auto`

## 7. Locking & Updates

- **`govard lock`**: `generate`, `check`, `diff`
- **`govard self-update`**: Update Govard binaries
- **`govard upgrade`**: Native framework upgrade pipeline (`--version <v>`)
- **`govard tunnel`**: Public access tunnel via `cloudflared`

## 8. Project Management

- **`govard project list`**: List all projects
- **`govard project delete <name>`**: Remove project completely
- **`govard project clean`**: Clean up resources