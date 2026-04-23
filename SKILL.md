---
name: fastfony
description: Build a Symfony starter-kit using Fastfony bundles (https://fastfony.com). Use this skill when the user mentions Fastfony, wants to bootstrap a Symfony starter-kit or boilerplate, integrate `fastfony/identity-bundle`, or add/customize authentication (form login, magic login link, password reset, user/role/group management) in a Symfony app via Fastfony. Also triggers on phrases like "Symfony starter-kit", "Symfony boilerplate", "vibe code a Symfony project". Do not use this skill for generic Symfony questions unrelated to Fastfony bundles.
---

# Fastfony — Symfony starter-kit skill

Fastfony is a collection of focused, composable Symfony bundles. Rather than one monolithic starter-kit, a developer picks the bundles they need and assembles their own boilerplate.

Use this skill to help a developer:

1. **Bootstrap** a new Symfony starter-kit using Fastfony bundles.
2. **Integrate** a Fastfony bundle into an existing Symfony app.
3. **Customize** a Fastfony bundle (templates, config, entities).

## Available bundles

| Bundle | Purpose | License | Reference |
|---|---|---|---|
| [`fastfony/identity-bundle`](https://github.com/fastfony/identity-bundle) | Form login, magic login link, registration, password reset, users/roles/groups | MIT | [references/identity-bundle.md](references/identity-bundle.md) |

**Only `identity-bundle` exists today.** The ecosystem is growing. If the user asks about a bundle not listed here, say so explicitly and offer to check [github.com/fastfony](https://github.com/fastfony). Do not invent bundles.

## Available packs

| Pack | Purpose | License | Reference |
|---|---|---|---|
| [`fastfony/quality-pack`](https://github.com/fastfony/quality-pack) | PHPStan + PHP-CS-Fixer + Twig-CS-Fixer + CI workflow + Makefile, in one `composer require --dev` | MIT | [references/bootstrap-project.md §8](references/bootstrap-project.md) |

## Core philosophy

- **Composable over monolithic.** Each bundle has a single responsibility. **Never recommend the legacy `fastfony/fastfony` monolithic starter-kit** — it was intentionally split into bundles because it was too opinionated and complete to be reusable.
- **Modern frontend = AssetMapper only.** Unlike the legacy `fastfony/fastfony` (which used Webpack Encore + Vue), starter-kits generated with this skill **must use Symfony AssetMapper exclusively**. No Webpack Encore, no Vite, no JS bundlers, no Node build step. Stimulus and Turbo (via Symfony UX) are the supported interactivity layer. See [references/conventions.md#frontend--assets](references/conventions.md#frontend--assets).
- **Standard Symfony.** Config via `config/packages/*.yaml`, templates overridable via `templates/bundles/{BundleName}/`, database via Doctrine ORM. No proprietary patterns.
- **Timestampable + Managers.** Entities use Stof's Timestampable; database operations go through dedicated `*Manager` services (not repositories). See [references/conventions.md](references/conventions.md).

## Primary workflows

### 1. Bootstrap a new Symfony starter-kit

Read [references/bootstrap-project.md](references/bootstrap-project.md) first. Summary:

1. Ask the user: Symfony 7.4 LTS (PHP 8.2+) or Symfony 8.0 (PHP 8.4+)?
2. Create the skeleton: `symfony new ... --webapp --no-git`.
3. **Enable contrib**: `composer config extra.symfony.allow-contrib true`.
4. `git init` + initial commit.
5. `docker compose up -d` (Postgres + Mailpit).
6. Wire Tailwind via AssetMapper: `composer require symfonycasts/tailwind-bundle` → `tailwind:init` → `tailwind:build`.
7. Install `fastfony/identity-bundle` (Flex applies recipe cleanly because of step 3).
8. Migrate, create first user, smoke test `/login` + `/secure-area/`.
9. Optionally `composer require --dev fastfony/quality-pack` (PHPStan + PHP-CS-Fixer + Twig-CS-Fixer + GitHub Actions workflow + Makefile in one pack) — see [references/bootstrap-project.md §8](references/bootstrap-project.md).

### 2. Add `identity-bundle` to an existing Symfony app

Read [references/identity-bundle.md](references/identity-bundle.md) — step-by-step integration covering install, Stof config, security.yaml, routing, mailer, migrations, and first-user creation.

### 3. Customize `identity-bundle`

All customization paths (template overrides, bundle config, entity extension, email subjects/content) are in [references/identity-bundle.md](references/identity-bundle.md) under "Customization".

## Execution rules

- **Verify the environment first.** Before installing anything:
  - Ask which **Symfony version** the starter-kit targets (default 7.4 LTS if unclear).
  - Check the matching **PHP version**:
    - Symfony **7.4 LTS** → PHP ≥ **8.2**. Do **not** tell a 7.4 user they need PHP 8.4.
    - Symfony **8.0** → PHP ≥ **8.4**.
  - `symfony -v` (Symfony CLI) and `composer -V` present.
  - If a Homebrew machine has multiple PHPs (`/opt/homebrew/opt/php@X.Y/`), use PATH prefix or a `.php-version` file rather than `brew link --force`.

  If any requirement is missing, tell the user before proceeding.

- **Enable contrib recipes BEFORE any `composer require` of a Fastfony or Stof bundle.** Run `composer config extra.symfony.allow-contrib true` right after creating the project. If you skip this step, the Flex recipe is marked IGNORED, then "installed" — but `security.yaml` / `bundles.php` aren't patched. `recipes:install --force` does not recover the missing `add-lines`. This is a single most common way to waste 15 minutes debugging a half-configured app.

- **Read the reference.** Always read the relevant `references/*.md` file in full before running commands. Do not rely on your memory of the summary above.

- **Prefer `symfony console` / `symfony server` over raw `php bin/console`.** The Symfony CLI sometimes injects Docker-mapped ports from `compose.override.yaml`, but it's not guaranteed (`symfony var:export | grep DATABASE_URL` may return empty even with services running). Whenever the project's `.env` hardcodes `DATABASE_URL=...@127.0.0.1:5432/...`, also verify that no other Postgres container on the host is bound to fixed port 5432 — otherwise the project silently writes to that shared DB. See [references/troubleshooting.md](references/troubleshooting.md#tables-already-exist-in-a-fresh-project--doctrinemigrationsdiff-says-no-changes-detected).

- **One concern at a time.** Apply changes step by step: project skeleton → contrib enabled → docker up → tailwind → bundle install → config verify → DB → first user → smoke test. Run a relevant check after each step (`symfony console about`, `debug:router`, `debug:config security`). Do not batch unrelated changes.

- **Commit at milestones.** Suggest `git commit` after each completed step. The user can always squash later.

- **Confirm destructive actions.** `doctrine:schema:update --force`, migration rollbacks, or overwriting existing config files require explicit user confirmation.

- **Prefer Symfony Flex.** When Flex is available (and contrib is enabled), let it run recipes. Only fall back to manual configuration when the user says Flex is not installed, or when a recipe was IGNORED before contrib was enabled (see [references/troubleshooting.md](references/troubleshooting.md)).

## Do not

- Recommend the legacy `fastfony/fastfony` monolithic starter-kit.
- Install Webpack Encore, Vite, or any other JS bundler. AssetMapper is the only asset strategy.
- Introduce Vue, React, Svelte, or any SPA framework as a build-step dependency. If interactivity is needed, use Stimulus and Turbo (Symfony UX).
- Invent bundles or config options not documented in the reference files.
- Skip `security.yaml` configuration — `identity-bundle` cannot work without it.
- Run `doctrine:schema:update --force` in an app with existing data.
- Override templates or configuration without confirming the user wants to deviate from Fastfony defaults.

## Troubleshooting

Common issues (Flex recipe didn't run, login link email never arrives, `StofDoctrineExtensionsBundle` not found, etc.) are in [references/troubleshooting.md](references/troubleshooting.md).
