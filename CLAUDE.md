# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Fleetbase is a modular logistics and supply chain operating system. The repo is split into three main parts:

- `api/` — Laravel 10 PHP backend (PHP 8.0–8.2), served via Laravel Octane
- `console/` — Ember.js 5.4 (Octane edition) frontend, using Ember Engines for extension mounting
- `packages/` — Local source for shared packages (`core-api`, `ember-core`, `ember-ui`, `fleetops`, `fleetops-data`, `storefront`, `iam-engine`, `ledger`, `pallet`, `registry-bridge`, `dev-engine`)

The entire stack runs via Docker Compose. The standard ports are `8000` (API) and `4200` (Console).

## Commands

### Docker (primary local dev)

```bash
docker compose up                          # start all services
docker compose exec application bash      # shell into API container
docker compose exec application bash -c "php artisan <cmd>"
```

### API (`api/`)

```bash
composer install
php artisan migrate
php artisan test                           # run all tests
php artisan test --filter ExampleTest     # run a single test
./vendor/bin/phpunit --filter ExampleTest # alternative single test
php artisan key:generate --show           # generate APP_KEY if missing
```

### Console (`console/`)

Uses pnpm (v11). Run commands from the `console/` directory.

```bash
pnpm install
pnpm start                  # dev server on :4200
pnpm start:dev              # explicitly development environment
pnpm build                  # production build
pnpm test                   # lint + ember tests
pnpm test:ember             # ember tests only
pnpm lint                   # all linters
pnpm lint:fix               # auto-fix lint issues
pnpm lint:js                # ESLint only
pnpm lint:hbs               # template lint only
pnpm lint:css               # stylelint only
```

### Package Linker (local extension development)

When working on packages in `packages/`, use the linker to wire them into the console and API without publishing.

```bash
# From repo root — install as global CLI
npm link

flb-package-linker status
flb-package-linker list
flb-package-linker doctor

# Link FleetOps and shared packages, then install
flb-package-linker enable fleetops --shared ember-core ember-ui fleetops-data
flb-package-linker install

# Reset all local links
flb-package-linker reset && flb-package-linker install

# Or run without global install
node scripts/package-linker.mjs status
```

The linker updates `console/package.json`, `console/pnpm-workspace.yaml`, `console/.npmrc`, `api/composer.json`, and `.fleetbase-dev-links.json`.

## Architecture

### API: thin shell over Composer packages

`api/app/` contains almost no business logic — just a base `Controller.php`, a single `User.php` model, and Laravel's standard providers. All domain logic lives in Composer packages installed from the Fleetbase registry (`fleetbase/core-api`, `fleetbase/fleetops-api`, `fleetbase/storefront-api`, etc.). `api/composer.json` is the integration point where these packages are pinned and composed together. The only application-specific route is a `GET /health` endpoint in `RouteServiceProvider.php`.

### Console: extension-first Ember Engines architecture

The console is intentionally thin. Its main job is authentication, admin settings, and loading/mounting extensions. The extension lifecycle on boot is:

1. **Initializers** (run before app ready): `load-intl-polyfills` → `load-runtime-config` → `load-socketcluster-client`
2. **Instance initializers** (run after app ready):
   - `load-extensions` — calls `ExtensionManager.loadExtensions()` to fetch installed extensions from the API
   - `initialize-registries` — creates console-level service registries (`@fleetbase/console`, `auth:login`)
   - `initialize-widgets` — registers dashboard and widget definitions
   - `setup-extensions` — runs each extension's `extension.js` to mount Ember Engines and register routes/components

Extensions land as Ember Engines (e.g. `@fleetbase/fleetops-engine`, `@fleetbase/storefront-engine`) and mount under `console/:slug` via `virtual` catch-all routes.

### Runtime configuration

`console/fleetbase.config.json` is the runtime config file served as a static asset. On boot, `load-runtime-config` fetches this file and applies an allowlisted set of keys to the Ember `config/environment` object:

| Key | Effect |
|-----|--------|
| `API_HOST` | API base URL |
| `SOCKETCLUSTER_HOST/PORT/SECURE/PATH` | WebSocket connection |
| `OSRM_HOST` | Routing server |
| `EXTENSIONS` | Preloaded extension list |

In production this config is cached in `localStorage` for 1 hour. Changing only `fleetbase.config.json` (not rebuilding the app) is the intended way to reconfigure a deployed console.

### Docker services

| Service | Image | Purpose |
|---------|-------|---------|
| `application` | `fleetbase/fleetbase-api` | Main Laravel API (Octane) |
| `queue` | `fleetbase/fleetbase-api` | Laravel queue worker |
| `scheduler` | `fleetbase/fleetbase-api` | Cron via `go-crond` |
| `socket` | `socketcluster/socketcluster:v17.4.0` | Real-time WebSocket server |
| `database` | `mysql:8.0-oracle` | Primary database |
| `cache` | `redis:4-alpine` | Redis for queue and cache |
| `console` | Built from `console/Dockerfile` | Nginx-served Ember app |
| `httpd` | Built from `docker/httpd/Dockerfile` | Reverse proxy on :8000 |

Override environment variables via `docker-compose.override.yml` — an example is at `docker-compose.override.yml.example`.

### API deployment

`api/deploy.sh` is the canonical deploy sequence: `migrate`, `sandbox:migrate`, `fleetbase:seed`, `fleetbase:create-permissions`, `queue:restart`, `schedule-monitor:sync`, `cache:clear`, `route:clear`, `config:cache`, `route:cache`, `registry:init`.

### Extension development

A package is treated as a Fleetbase extension when its `package.json` has `fleetbase-extension` in `keywords` or it has an `extension.json`. Shared non-extension packages supported by the linker are `ember-core`, `ember-ui`, `fleetops-data`, and (backend-only) `core-api`.

The CI pipeline (`ci.yml`) uses Docker Bake (`docker-bake.hcl`) to build all images. A `composer-auth.json` is generated from `_GITHUB_AUTH_TOKEN` for private Composer packages.
