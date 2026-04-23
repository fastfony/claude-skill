# Bootstrap a new Symfony starter-kit with Fastfony

Goal: end up with a fresh Symfony project, ready-to-use authentication via `fastfony/identity-bundle`, and optional developer tooling. Commit at every milestone so the user can cherry-pick or revert.

## 0. Requirements

```bash
php -v            # ≥ 8.2
symfony -v        # Symfony CLI installed (recommended)
composer -V
```

If the Symfony CLI is missing, install from https://symfony.com/download.

**No Node / npm required.** Fastfony starter-kits use Symfony AssetMapper exclusively for frontend assets. Do not introduce Webpack Encore, Vite, or any bundler. See [conventions.md — Frontend](conventions.md#frontend--assets).

## 1. Create the project

Ask the user for the project name (used as the directory name and composer package name). Default: `my-starter`.

```bash
symfony new my-starter --version="7.4.*" --webapp
cd my-starter
```

`--webapp` pulls in the common webapp recipe (Twig, Doctrine, Security, Mailer, Form, Validator, **AssetMapper**, Stimulus, Turbo, etc.) — exactly what Fastfony starter-kits are built on. If they want a slimmer kit, use plain `symfony new my-starter --version="7.4.*"` and add pieces later, but **always keep AssetMapper as the asset handler** (never add Encore or a JS bundler).

## 2. Initial commit

```bash
git init
git add .
git commit -m "chore: initial Symfony 7.4 webapp skeleton"
```

## 3. Configure the database

Edit `.env.local` (create if needed) with the appropriate `DATABASE_URL`:

```
DATABASE_URL="postgresql://app:app@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
```

If the user wants Docker locally, the webapp recipe already includes a `compose.yaml` with Postgres. Start it:

```bash
docker compose up -d
```

Create the DB:

```bash
php bin/console doctrine:database:create
```

Commit:

```bash
git add .env .env.local compose.yaml compose.override.yaml 2>/dev/null
git commit -m "chore: configure database connection"
```

## 4. Configure the mailer

Edit `.env.local`:

```
MAILER_DSN=smtp://localhost:1025      # Mailpit / MailHog for local
# or null://null during initial bring-up (login link emails won't work)
```

For production, use a transactional provider DSN (SES, Mailgun, Postmark, etc.). See https://symfony.com/doc/current/mailer.html#transport-setup.

## 5. Install `fastfony/identity-bundle`

Switch to [references/identity-bundle.md](identity-bundle.md) and follow sections 2–7. Summary of what you'll do:

1. `composer require fastfony/identity-bundle`
2. If Flex ran: move to step 5. If not: manually register bundles, add configs.
3. Generate and run the first migration.
4. Create the first user with `php bin/console fastfony:user:create`.
5. Start the server (`symfony serve -d`) and log in at `/login`.

Commit:

```bash
git add .
git commit -m "feat: integrate fastfony/identity-bundle"
```

## 6. (Optional) Developer tooling

Only apply this step if the user wants it. The original Fastfony starter-kit uses a curated set; mirror whatever the user cares about.

### PHPStan

```bash
composer require --dev phpstan/phpstan phpstan/extension-installer phpstan/phpstan-symfony phpstan/phpstan-doctrine
```

`phpstan.dist.neon`:

```neon
includes:
    - vendor/phpstan/phpstan-symfony/extension.neon
    - vendor/phpstan/phpstan-doctrine/extension.neon

parameters:
    level: 6
    paths:
        - src
        - tests
    symfony:
        containerXmlPath: var/cache/dev/App_KernelDevDebugContainer.xml
```

Run: `vendor/bin/phpstan analyse`

### PHP-CS-Fixer

```bash
composer require --dev friendsofphp/php-cs-fixer
```

`.php-cs-fixer.dist.php`:

```php
<?php

$finder = (new PhpCsFixer\Finder())
    ->in([__DIR__ . '/src', __DIR__ . '/tests']);

return (new PhpCsFixer\Config())
    ->setRules([
        '@Symfony' => true,
        '@Symfony:risky' => true,
        'declare_strict_types' => true,
        'ordered_imports' => true,
    ])
    ->setRiskyAllowed(true)
    ->setFinder($finder);
```

Run: `vendor/bin/php-cs-fixer fix`

### PHPUnit

Already included in `--webapp`. Run `php bin/phpunit`.

Commit:

```bash
git add .
git commit -m "chore: add PHPStan + PHP-CS-Fixer"
```

## 7. (Optional) CI

Minimal `.github/workflows/ci.yaml`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: app
          POSTGRES_USER: app
          POSTGRES_DB: app_test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.3', coverage: none }
      - run: composer install --prefer-dist --no-progress
      - run: vendor/bin/phpstan analyse --no-progress
      - run: php bin/console doctrine:migrations:migrate --no-interaction --env=test
      - run: php bin/phpunit
        env:
          DATABASE_URL: postgresql://app:app@127.0.0.1:5432/app_test
```

## 8. Next bundles

As Fastfony grows, add new bundles the same way: `composer require fastfony/<name>-bundle`, then follow the bundle's reference file in this skill. Commit each bundle separately.

## Milestone checklist

- [ ] Symfony skeleton installed, initial commit
- [ ] Database connected, first migration runs
- [ ] Mailer DSN configured (real or dev)
- [ ] `identity-bundle` installed and configured
- [ ] First user created, login flow works end-to-end
- [ ] (Optional) Dev tooling + CI
