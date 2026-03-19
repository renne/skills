---
name: contributing
description: Guide for contributing effectively to LimeSurvey (https://github.com/LimeSurvey/LimeSurvey). Covers the PHP backend, JavaScript/Vue/React frontend, Twig templating, SCSS/Less/Stylus styling pipeline, OpenAPI/Swagger REST API docs, GitHub Actions CI, Cypress E2E testing, and cross-DB compatibility (MySQL, PostgreSQL, MSSQL). Use when writing patches, new features, reviewing PRs, or setting up a local dev environment for LimeSurvey.
---
# Contributing to LimeSurvey

Sources:
- Repository: https://github.com/LimeSurvey/LimeSurvey
- CI testing repo: https://github.com/LimeSurvey/limesurvey-core-testing
- Manual: https://manual.limesurvey.org/

## Overview

LimeSurvey is an open-source survey platform. The main repository is a PHP + JavaScript monorepo with a significant theming layer, REST API, and cross-database support.

**Language composition** (by bytes, descending):
- JavaScript — 15.9 MB (frontend logic, admin UI packages)
- PHP — 13.6 MB (core application, Yii-based MVC)
- CSS — 4.8 MB
- SCSS — 875 KB (theming/styling pipeline)
- Twig — 819 KB (template engine for themes)
- HTML, Less, Stylus, Vue, Shell, Makefile, …

**Default branch:** `master` (bug fixes) — new features go to `develop`.

---

## Repository Layout (key paths)

```
application/         # PHP application core (Yii MVC)
  commands/          # CLI console commands
  config/            # config-sample-mysql.php, config-sample-pgsql.php, config-sample-sqlsrv.php
  controllers/       # MVC controllers
  models/            # Active Record models
  views/             # PHP/Twig view templates
assets/
  packages/
    adminbasics/     # Admin UI package (JS; built with yarn)
    adminsidepanel/  # Admin side panel package (WIP)
themes/              # Survey & admin themes (Twig + SCSS/Less/Stylus)
tests/
  functional/        # PHPUnit functional tests (needs running app + Selenium/Firefox)
  unit/              # PHPUnit unit tests (DB matrix: MySQL/PostgreSQL/MSSQL)
  bin/               # Lint scripts, coverage checker
docs/
  PULL_REQUEST_TEMPLATE   # PR checklist template
  contributions/          # Logos/assets for contributor credits
  open-api/v1.json        # OpenAPI 3.x spec (388 KB, machine-generated)
  swagger/                # Legacy Swagger artifacts
  themes/                 # Theme development docs & example question theme
  limesurvey-ruleset.xml  # PHP_CodeSniffer ruleset
  release_notes.txt       # Full release history (~791 KB)
.github/workflows/
  main.yml           # PHP CI: functional + unit tests (PHP 7.4/8.3, MySQL/PostgreSQL/MSSQL)
  frontend.yml       # JS CI: yarn install + build + unit tests for adminbasics
```

---

## Branch Strategy

| Branch target | When to use |
|---------------|-------------|
| `master` | Bug fixes for already-released features |
| `develop` | New features and larger refactors |

PRs must reference a **Mantis issue** from [bugs.limesurvey.org](https://bugs.limesurvey.org). For minor internal changes ("Dev:" prefix) no Mantis issue is required. See the PR template at [`docs/PULL_REQUEST_TEMPLATE`](https://github.com/LimeSurvey/LimeSurvey/blob/master/docs/PULL_REQUEST_TEMPLATE).

---

## PHP Backend

- Framework: **Yii 1.x** (legacy MVC; class-based controllers, Active Record models).
- Supported PHP versions: **7.4** and **8.3** (both tested in CI).
- Coding standards: PHP_CodeSniffer with the project ruleset at [`docs/limesurvey-ruleset.xml`](https://github.com/LimeSurvey/LimeSurvey/blob/master/docs/limesurvey-ruleset.xml). Run via `composer test`.
- Static analysis: **Psalm** (`psalm-all.xml`) with low strictness, run in the `code-check` CI job.
- Autoloader sanity check: `php tests/check_autoloader.php` before `composer install`.

### Local setup (PHP)

```bash
composer install -vvv
# Choose the right sample config for your DB:
cp application/config/config-sample-mysql.php application/config/config.php
# (or config-sample-pgsql.php / config-sample-sqlsrv.php)

touch enabletests   # required to enable test mode
php application/commands/console.php install admin password MyLS no@email.com verbose
```

### Running PHP tests

```bash
# Unit tests (needs a running DB)
php vendor/bin/phpunit --testdox --stop-on-failure tests/unit

# Functional tests (needs Apache + Selenium/Firefox)
php vendor/bin/phpunit --testdox --stop-on-failure tests/functional

# Code quality (CodeSniffer + MessDetector + Psalm)
composer test
./vendor/bin/psalm -c psalm-all.xml
```

---

## JavaScript Frontend

The admin UI is split into Yarn workspaces under `assets/packages/`:

- **`adminbasics`** — primary admin package; has its own `package.json`, build, and test suite.
- **`adminsidepanel`** — side panel (currently disabled in CI; WIP).

Node.js **20.x** is required (CI uses 20.19.0).

### Local setup (JS)

```bash
yarn install                                        # root workspace
yarn --cwd ./assets/packages/adminbasics           # install adminbasics
yarn --cwd ./assets/packages/adminbasics build     # production build
yarn --cwd ./assets/packages/adminbasics run test  # unit tests
```

### Frontend CI workflow

Defined in [`.github/workflows/frontend.yml`](https://github.com/LimeSurvey/LimeSurvey/blob/master/.github/workflows/frontend.yml):
1. `composer install` (PHP deps required even for JS build).
2. `yarn install` (root).
3. Install + build `adminbasics`.
4. Run `adminbasics` tests.

---

## Theming & Templating

LimeSurvey themes live under `themes/` and use **Twig** for HTML templating combined with **SCSS**, **Less**, or **Stylus** for stylesheets.

- Survey themes control the look of the survey-taker interface.
- Admin themes control the backend UI.
- Example question theme structure: [`docs/themes/questiontheme_example/`](https://github.com/LimeSurvey/LimeSurvey/tree/master/docs/themes/questiontheme_example).
- Theme docs: [`docs/themes/`](https://github.com/LimeSurvey/LimeSurvey/tree/master/docs/themes).

When modifying themes:
- Edit `.twig` template files for markup changes.
- Edit `.scss`/`.less`/`.styl` files for style changes; compile them as part of the asset build.
- Run `chmod -R 777 ./themes` to ensure Apache can write to the themes directory during development.

---

## REST API (OpenAPI / Swagger)

- The OpenAPI v1 spec is committed as [`docs/open-api/v1.json`](https://github.com/LimeSurvey/LimeSurvey/blob/master/docs/open-api/v1.json) (~388 KB; OpenAPI 3.x format).
- Legacy Swagger artifacts are in [`docs/swagger/`](https://github.com/LimeSurvey/LimeSurvey/tree/master/docs/swagger).
- When adding or modifying API endpoints, update `docs/open-api/v1.json` to keep the spec in sync.
- The spec can be viewed interactively with any Swagger UI / Redoc viewer by pointing it at `v1.json`.

---

## CI / GitHub Actions

Two workflows run on all branches and PRs:

### `main.yml` — PHP CI ([source](https://github.com/LimeSurvey/LimeSurvey/blob/master/.github/workflows/main.yml))

| Job | What it does |
|-----|-------------|
| `CI-pipeline` | Functional tests: installs PHP 7.4 & 8.3, sets up Apache + Selenium (Firefox headless), runs `tests/functional` against MySQL |
| `unit-tests` | Unit tests: matrix of PHP 7.4/8.3 × MySQL 8.0/PostgreSQL 14/MSSQL 2022; uploads `cov.xml` as artifact |
| `test-coverage` | Downloads `cov.xml` and checks minimum coverage threshold (37%) via `tests/bin/check_coverage.php` |
| `code-check` | Runs `composer test` (CodeSniffer + MessDetector) and Psalm on `application/` |

### `frontend.yml` — JS CI ([source](https://github.com/LimeSurvey/LimeSurvey/blob/master/.github/workflows/frontend.yml))

| Job | What it does |
|-----|-------------|
| `job_build_packages` | Installs PHP deps (composer), installs and builds `adminbasics` |
| `job_run_tests` | Runs `adminbasics` unit tests |

Branch naming conventions used in CI triggers: `bug/**`, `feature/**`, `dev/**`, `zoho/**`.

---

## E2E Testing (Cypress)

LimeSurvey E2E tests live in a **separate repository**: [`LimeSurvey/limesurvey-core-testing`](https://github.com/LimeSurvey/limesurvey-core-testing) (default branch: `develop`, last active 2025-05-05).

- Tests are written in **Cypress**.
- They target the LimeSurvey core application running locally or in CI.
- When contributing features or bug fixes that affect user-facing flows, check whether Cypress tests in that repo need to be updated alongside the main PR.

---

## Database Compatibility

LimeSurvey supports three database engines. All unit tests run against all three in CI:

| DB | Version in CI | Config sample |
|----|--------------|---------------|
| MySQL | 8.0 | `application/config/config-sample-mysql.php` |
| PostgreSQL | 14 | `application/config/config-sample-pgsql.php` |
| MSSQL (SQL Server) | 2022 | `application/config/config-sample-sqlsrv.php` |

**DB compatibility rules for contributors:**
- Use Yii's Active Record / query builder instead of raw SQL wherever possible.
- If raw SQL is unavoidable, test on all three DBs (CI will catch regressions).
- Schema migrations must be database-agnostic.
- MSSQL requires `InnoDB`-equivalent settings via `DBENGINE=INNODB` env var in CI.

---

## Typical Contribution Workflow

1. **Find or create a Mantis issue** at [bugs.limesurvey.org](https://bugs.limesurvey.org).
2. **Fork** the repo and check out the appropriate branch (`master` for bugfixes, `develop` for features).
3. **Make changes**, following:
   - PHP: CodeSniffer ruleset in `docs/limesurvey-ruleset.xml`
   - JS: project-specific ESLint/Prettier config in `assets/packages/adminbasics`
   - Templates: Twig syntax, consistent with existing theme structure
4. **Run tests locally** (PHP unit/functional + JS unit).
5. **Open a PR** targeting the correct branch. Use the PR template (`docs/PULL_REQUEST_TEMPLATE`).
6. **CI will automatically run** `main.yml` (PHP 7.4+8.3 / MySQL+PostgreSQL+MSSQL) and `frontend.yml`.
7. For user-facing changes, also update or add Cypress tests in [`limesurvey-core-testing`](https://github.com/LimeSurvey/limesurvey-core-testing).

---

## Known Quirks and Limitations

- ⚠️ The `adminsidepanel` build is **disabled in CI** (commented out in `frontend.yml`) — building it locally may fail.
- ⚠️ Functional tests require a running **Apache + Selenium (Firefox headless)** stack. Unit tests are standalone (no browser).
- ⚠️ `touch enabletests` must exist in the project root before running any tests.
- ⚠️ `chmod -R 777 ./tmp ./upload ./themes ./application/config` is required for Apache to write to these directories during functional tests.
- LimeSurvey uses the **Mantis** bug tracker (not GitHub Issues) for tracking bugs and features. The main repo may appear to have issues disabled on GitHub.
- The `docs/contributions/` folder only contains logo images (BrowserStack, Scrutinizer, TravisCI) — no written contributor guide exists at that path; contributor guidance is on the manual wiki.
- Release history is maintained as a single large text file: `docs/release_notes.txt` (~791 KB).

---

## References

- [LimeSurvey/LimeSurvey GitHub Repository](https://github.com/LimeSurvey/LimeSurvey)
- [LimeSurvey/limesurvey-core-testing (Cypress tests)](https://github.com/LimeSurvey/limesurvey-core-testing)
- [How to contribute new features (manual)](https://manual.limesurvey.org/How_to_contribute_new_features)
- [LimeSurvey release schedule (manual)](https://manual.limesurvey.org/LimeSurvey_roadmap#Current_release_schedule)
- [Mantis bug tracker](https://bugs.limesurvey.org)
- [OpenAPI v1 spec](https://github.com/LimeSurvey/LimeSurvey/blob/master/docs/open-api/v1.json)
- [PHP CodeSniffer ruleset](https://github.com/LimeSurvey/LimeSurvey/blob/master/docs/limesurvey-ruleset.xml)
- [PR template](https://github.com/LimeSurvey/LimeSurvey/blob/master/docs/PULL_REQUEST_TEMPLATE)
- [GitHub Actions — main.yml](https://github.com/LimeSurvey/LimeSurvey/blob/master/.github/workflows/main.yml)
- [GitHub Actions — frontend.yml](https://github.com/LimeSurvey/LimeSurvey/blob/master/.github/workflows/frontend.yml)
