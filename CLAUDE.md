# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in
this repository.

## Project Overview

This is a Doodba-based Odoo 18.0 deployment project for Vikingo Software. Doodba is a
Docker-based development and deployment framework for Odoo that uses git-aggregator to
manage multiple addon repositories.

**Key characteristics:**

- Odoo Version: 18.0
- Database: PostgreSQL 17
- Deployment framework: Doodba (Tecnativa)
- Template: doodba-copier-template v9.0.0
- Production URL: gestion.vikingosoftware.com
- Initial language: Spanish (Argentina)

## Architecture

### Directory Structure

```
├── odoo/
│   ├── custom/
│   │   ├── src/           # Git-aggregated addon repositories
│   │   │   ├── odoo/      # Core Odoo (from OCA OCB or official repo)
│   │   │   ├── private/   # Custom private addons
│   │   │   ├── repos.yaml # Repository aggregation config
│   │   │   └── addons.yaml # Addon installation config
│   │   ├── conf.d/        # Odoo configuration files
│   │   ├── dependencies/  # Python/apt dependencies
│   │   ├── build.d/       # Docker build scripts
│   │   ├── entrypoint.d/  # Container startup scripts
│   │   └── ssh/           # SSH keys for git access
│   └── auto/              # Auto-generated addons symlinks
├── tasks.py               # Invoke-based task runner (main CLI)
├── common.yaml            # Base docker-compose config
├── devel.yaml             # Development environment config
├── test.yaml              # Testing environment config
├── prod.yaml              # Production environment config
└── setup-devel.yaml       # Git aggregation setup config
```

### Repository Management

This project uses **git-aggregator** to pull Odoo core and community addons from
multiple repositories. The configuration is in `odoo/custom/src/repos.yaml`:

- Each repository is cloned into `odoo/custom/src/`
- Repositories can have multiple remotes and merge specific branches/PRs
- Use shallow clones by default for speed (`$DEPTH_DEFAULT=1`)
- Addons from repos are selectively enabled in `odoo/custom/src/addons.yaml`

**Current repositories:**

- OCA: OpenUpgrade, spreadsheet, web, server-ux, server-auth, server-backend,
  server-brand, account-financial-tools, account-reconcile, reporting-engine,
  account-financial-reporting, queue, helpdesk, stock-logistics-transport
- INGADHOC: odoo-argentina-ce (localization)
- nanosgr: vikingo_web_responsive

### Docker Architecture

The project uses multiple Docker Compose configurations:

- `common.yaml` - Base service definitions (odoo, db, smtpfake)
- `devel.yaml` - Development with hot-reload, debugging, and helper services
- `test.yaml` - Testing environment
- `prod.yaml` - Production deployment

Development services include:

- `odoo` - Main Odoo service with debugpy support
- `odoo_proxy` - Whitelist proxy for development
- `db` - PostgreSQL with autoconf
- `pgweb` - Database web UI (port 18081)
- `smtp` - MailHog fake SMTP (port 18025)
- `wdb` - Web debugger (port 18984)
- Proxy services for external resources (Google Fonts, CDNs, etc.)

**Port mappings (development):**

- 18069 - Odoo web interface
- 18072 - Odoo longpolling
- 18081 - PgWeb database UI
- 18025 - MailHog SMTP UI
- 18984 - WDB debugger
- 18899 - Debugpy Python debugger

## Common Commands

All commands use **Invoke** (`invoke` or `inv` command) defined in `tasks.py`. Run from
project root.

### Initial Setup

```bash
# First-time setup: create directory structure, pre-commit hooks
invoke develop

# Download/update all git repositories defined in repos.yaml
invoke git-aggregate
```

### Docker Operations

```bash
# Build Docker images
invoke img-build
invoke img-build --no-pull  # Skip pulling base images

# Pull pre-built images
invoke img-pull

# Start environment (detached, with auto-reload)
invoke start

# Start with Python debugger enabled
invoke start --debugpy

# Stop environment
invoke stop

# Stop and purge (remove containers, volumes, images)
invoke stop --purge

# Restart Odoo containers
invoke restart
```

### Development Workflow

```bash
# Install addon from current directory
invoke install restart

# Install specific addon(s)
invoke install --modules=sale,purchase restart

# Install all addons from a category
invoke install --private restart      # Private addons
invoke install --extra restart        # Extra addons
invoke install --core restart         # Core addons

# Uninstall addon from current directory
invoke uninstall

# Uninstall specific addon(s)
invoke uninstall --modules=addon_name
```

### Testing

```bash
# Run tests for addon in current directory
invoke test

# Run tests for specific addon(s)
invoke test --modules=sale,purchase

# Run tests with Python debugger
invoke test --debugpy

# Test mode options
invoke test --mode=init    # Default: install and test
invoke test --mode=update  # Update and test

# Skip specific modules during testing
invoke test --modules=base,sale --skip=sale
```

### Database Operations

```bash
# Reset database with base modules
invoke resetdb

# Reset with specific modules
invoke resetdb --modules=sale,purchase

# Reset with all private addons
invoke resetdb --private

# Reset with addon dependencies only
invoke resetdb --modules=my_addon --dependencies

# Create database snapshot
invoke snapshot  # Creates devel-YYYY_MM_DD-HH_MM

# Restore latest snapshot
invoke restore-snapshot

# Restore specific snapshot
invoke restore-snapshot --snapshot-name=devel-2024_01_27-10_30
```

### Translation Management

```bash
# Update .pot/.po files for addon in current directory
invoke updatepot

# Update for specific module
invoke updatepot --module=sale

# Update all modules in a repository
invoke updatepot --repo=private

# Update all modules (slow)
invoke updatepot --all

# Options
invoke updatepot --no-msgmerge           # Don't merge .pot into .po files
invoke updatepot --fuzzy-matching        # Use fuzzy matching when merging
invoke updatepot --no-remove-dates       # Keep dates in .po files
```

### Module Scaffolding

```bash
# Create new module in current directory
invoke scaffold --module-name=my_new_module

# Create in specific path
invoke scaffold --module-name=my_module --path=odoo/custom/src/private
```

### Logs and Debugging

```bash
# View logs (last 10 lines, follow mode)
invoke logs

# View more lines
invoke logs --tail=100

# View specific container logs
invoke logs --container=odoo
invoke logs --container=db,smtp
```

### Code Quality

```bash
# Run all pre-commit hooks (linting, formatting)
invoke lint

# Run with verbose output
invoke lint --verbose
```

### Workspace Management

```bash
# Regenerate VSCode workspace file
invoke write-code-workspace-file

# This is automatically called by develop and git-aggregate
```

## Development Practices

### Adding New Repositories

1. Edit `odoo/custom/src/repos.yaml` to add the repository configuration
2. Edit `odoo/custom/src/addons.yaml` to specify which addons to install (use `-'*'` to
   install all)
3. Run `invoke git-aggregate` to download the repository
4. Run `invoke install --modules=new_addon restart` to install

### Creating Custom Addons

Custom addons should go in `odoo/custom/src/private/`:

```bash
cd odoo/custom/src/private
invoke scaffold --module-name=my_custom_addon
```

### Working with Branches and PRs

To merge a specific PR from a repository, edit `repos.yaml`:

```yaml
./web:
  defaults:
    depth: $DEPTH_MERGE # Use deeper history for PR merging
  remotes:
    web: https://github.com/OCA/web.git
  target: web $ODOO_VERSION
  merges:
    - web $ODOO_VERSION
    - web refs/pull/1234/head # Merge PR #1234
```

Then run `invoke git-aggregate`.

### Environment Variables

Key environment variables (set in docker-compose files or shell):

- `DOODBA_ENVIRONMENT` - Environment name (devel/test/prod)
- `DOODBA_DEBUGPY_ENABLE` - Enable Python debugger (0/1)
- `DOODBA_WITHOUT_DEMO` - Skip demo data (false/all)
- `DOODBA_UID` / `DOODBA_GID` - User/group IDs for file permissions
- `ODOO_VERSION` - Odoo version (extracted from common.yaml)

### VSCode Integration

The project auto-generates a `.code-workspace` file with:

- Python debugging configuration (debugpy on port 18899)
- Browser debugging (Firefox, Chrome)
- Tasks for start/stop/restart/install/test operations
- Proper path mappings for all addon folders

### Pre-commit Hooks

The project uses pre-commit for code quality:

- Ruff (Python linting and formatting)
- Prettier (JSON, YAML, Markdown, XML formatting)
- ESLint (JavaScript linting)
- Pylint with odoo-specific checks
- OCA-specific checks for Odoo modules

Hooks run automatically on commit, or manually via `invoke lint`.

## Important Notes

- **Never edit `common.yaml` for environment-specific config** - use devel.yaml,
  test.yaml, or prod.yaml
- **The `docker-compose.yml` symlink** points to devel.yaml by default
- **Database name** is "devel" in development, "prod" in production
- **Custom addons** must be in `odoo/custom/src/private/` to be properly tracked
- **The `odoo/auto/` directory** is auto-generated and git-ignored - don't edit manually
- **Python dependencies** go in `odoo/custom/dependencies/pip.txt`
- **System dependencies** go in `odoo/custom/dependencies/apt.txt`
- **After updating repos.yaml**, always run `invoke git-aggregate` to apply changes
- **After installing/updating modules**, use `invoke restart` to reload Odoo
- **Tests must be run with workers=0** (automatically set by invoke test)

## Odoo Version Specifics

This project uses Odoo 18.0:

- Python 3 is required (Python 3.10+)
- Modern ORM features available
- Asset handling uses new system
- For v19+ resetdb behavior changes (uses native CLI instead of click-odoo-initdb)
