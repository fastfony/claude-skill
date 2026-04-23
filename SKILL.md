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

## Core philosophy

- **Composable over monolithic.** Each bundle has a single responsibility. **Never recommend the legacy `fastfony/fastfony` monolithic starter-kit** — it was intentionally split into bundles because it was too opinionated and complete to be reusable.
- **Modern frontend = AssetMapper only.** Unlike the legacy `fastfony/fastfony` (which used Webpack Encore + Vue), starter-kits generated with this skill **must use Symfony AssetMapper exclusively**. No Webpack Encore, no Vite, no JS bundlers, no Node build step. Stimulus and Turbo (via Symfony UX) are the supported interactivity layer. See [references/conventions.md#frontend--assets](references/conventions.md#frontend--assets).
- **Standard Symfony.** Config via `config/packages/*.yaml`, templates overridable via `templates/bundles/{BundleName}/`, database via Doctrine ORM. No proprietary patterns.
- **Timestampable + Managers.** Entities use Stof's Timestampable; database operations go through dedicated `*Manager` services (not repositories). See [references/conventions.md](references/conventions.md).

## Primary workflows

### 1. Bootstrap a new Symfony starter-kit

Read [references/bootstrap-project.md](references/bootstrap-project.md) first. Summary:

1. Create a Symfony skeleton with the webapp recipe.
2. `git init` and initial commit.
3. Install the first Fastfony bundle (usually `identity-bundle`).
4. Follow the integration guide for that bundle.
5. Optionally add dev tooling (PHPStan, PHP-CS-Fixer, PHPUnit) per [references/conventions.md](references/conventions.md).

### 2. Add `identity-bundle` to an existing Symfony app

Read [references/identity-bundle.md](references/identity-bundle.md) — step-by-step integration covering install, Stof config, security.yaml, routing, mailer, migrations, and first-user creation.

### 3. Customize `identity-bundle`

All customization paths (template overrides, bundle config, entity extension, email subjects/content) are in [references/identity-bundle.md](references/identity-bundle.md) under "Customization".

## Execution rules

- **Verify the environment first.** Before installing anything, check:
  - Symfony ≥ 7.4: `composer show symfony/framework-bundle`
  - PHP ≥ 8.2: `php -v`
  - Symfony Flex present: `composer show symfony/flex`

  If any requirement is missing, tell the user before proceeding.

- **Read the reference.** Always read the relevant `references/*.md` file in full before running commands. Do not rely on your memory of the summary above.

- **One concern at a time.** Apply changes step by step: bundle install → config → routing → security → DB schema → first user. Run `php bin/console about` or another relevant check after each step. Do not batch unrelated changes.

- **Commit at milestones.** Suggest `git commit` after each completed step (install, config, DB, first user). The user can always squash later.

- **Confirm destructive actions.** `doctrine:schema:update --force`, migration rollbacks, or overwriting existing config files require explicit user confirmation.

- **Prefer Symfony Flex.** When Flex is available, let it run recipes. Only fall back to manual configuration when the user says Flex is not installed or when they've opted out.

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
