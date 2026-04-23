# Bootstrap a new Symfony starter-kit with Fastfony

Goal: end up with a fresh Symfony project, ready-to-use authentication via `fastfony/identity-bundle`, Tailwind wired through AssetMapper, and optional developer tooling. Commit at every milestone so the user can cherry-pick or revert.

## 0. Requirements

First decide which Symfony version the starter-kit targets. Ask the user if unclear; default to **7.4 LTS**.

| Symfony | PHP minimum | Why pick it |
|---|---|---|
| **7.4 LTS** | **8.2** | 3-year LTS window, maximum ecosystem compat |
| **8.0** | **8.4** | Cutting-edge, but requires a very recent PHP |

⚠️ Don't tell the user "PHP 8.4 is required" if they're going with Symfony 7.4 — 7.4 works on PHP 8.2+. Version the requirement to the Symfony release.

Verify:

```bash
php -v            # must satisfy the PHP minimum of the chosen Symfony version
symfony -v        # Symfony CLI — install from https://symfony.com/download if missing
composer -V
```

### Multiple PHP versions (macOS / Homebrew)

If the user needs a PHP newer than their system default (common case: Symfony 8 on a machine whose `php` is 8.2), each Homebrew PHP lives at `/opt/homebrew/opt/php@X.Y/bin/php`. Check:

```bash
ls /opt/homebrew/opt/ | grep php@
```

Per-project switch:

1. Create `.php-version` in the project root with the major.minor (e.g. `8.4`). Symfony CLI (`symfony console`, `symfony server`) reads it automatically.
2. For raw `composer` / `php bin/console` calls, prefix PATH inline:
   ```bash
   PATH="/opt/homebrew/opt/php@8.4/bin:$PATH" composer install
   ```

Do **not** `brew link --force php@8.4` — that changes the user's system-wide PHP and can break their other projects.

### No Node / npm

Fastfony starter-kits use Symfony AssetMapper exclusively for frontend assets. Do not introduce Webpack Encore, Vite, or any bundler. See [conventions.md — Frontend](conventions.md#frontend--assets).

## 1. Create the project

Ask for the project directory name (default `my-starter`), then run:

```bash
cd <parent-dir>
# Symfony 7.4 LTS (PHP 8.2+):
symfony new my-starter --version="7.4.*" --webapp --no-git

# OR Symfony 8.0 (PHP 8.4+):
PATH="/opt/homebrew/opt/php@8.4/bin:$PATH" symfony new my-starter --version="8.0.*" --webapp --no-git

cd my-starter
```

`--webapp` pulls in the common recipe (Twig, Doctrine, Security, Mailer, Form, Validator, **AssetMapper**, Stimulus, Turbo) — exactly what Fastfony kits are built on. `--no-git` lets us init git ourselves after the contrib config step (§3).

If the Symfony 8 project was created with a non-default PHP, write the `.php-version` file immediately:

```bash
echo "8.4" > .php-version
```

## 2. Enable contrib recipes (do this BEFORE any `composer require`)

**Critical, easy-to-miss step.** Fastfony bundles (and stof/doctrine-extensions-bundle) live in `symfony/recipes-contrib`. Their Flex recipes are **IGNORED** by default. Once ignored, `composer recipes:install --force` only replays `copy-from-recipe`, not `add-lines` operations on `security.yaml` / `bundles.php` — leaving the app half-configured.

Enable contrib **before** the first `composer require` of any Fastfony bundle:

```bash
composer config extra.symfony.allow-contrib true
```

If the user already ran a contrib-requiring bundle without this, see [troubleshooting.md — Contrib recipe ignored](troubleshooting.md#contrib-recipe-ignored--manual-config-needed).

## 3. Initial commit

```bash
git init
git add .
git commit -m "chore: initial Symfony <version> webapp skeleton"
```

## 4. Start Docker services (DB + mailer)

The webapp recipe ships `compose.yaml` (Postgres) and `compose.override.yaml` (Mailpit mail catcher with dynamic port mapping — `ports: "5432"` instead of `"5432:5432"`). Nothing to edit: the Symfony CLI auto-detects the host ports and injects `DATABASE_URL` / `MAILER_DSN` when you use `symfony console` / `symfony server`.

```bash
docker compose up -d
docker compose ps        # verify healthy
```

Then always call commands via `symfony console ...` (not `php bin/console ...`) so dynamic ports propagate.

Mailpit UI: check `docker compose ps` for the 8025 host port (random). Open it in the browser to see captured emails (login links, password resets).

## 5. Wire up Tailwind via AssetMapper

```bash
composer require symfonycasts/tailwind-bundle
symfony console tailwind:init       # downloads the Tailwind v4 binary, adds @import "tailwindcss"; to assets/styles/app.css
```

**Do not add `{{ tailwind_stylesheet() }}` to `base.html.twig`** — that helper existed in older versions. With `symfonycasts/tailwind-bundle ≥ 0.11` + Tailwind v4, the integration flows entirely through AssetMapper:

- `assets/app.js` imports `./styles/app.css`
- `base.html.twig` already has `{{ importmap('app') }}` in its javascripts block, which emits both the JS and the CSS link tags

Clean the placeholder `body { background-color: skyblue; }` rule out of `assets/styles/app.css`; leave only `@import "tailwindcss";` (plus any custom rules).

Build the CSS once (dev):

```bash
symfony console tailwind:build
```

For live rebuilding during development, use a second terminal:

```bash
symfony console tailwind:build --watch
```

Commit:

```bash
git add .
git commit -m "feat: add symfonycasts/tailwind-bundle (Tailwind v4 via AssetMapper)"
```

## 6. Install `fastfony/identity-bundle`

Switch to [references/identity-bundle.md](identity-bundle.md) and follow it end-to-end. Summary of what you'll do:

1. `composer require fastfony/identity-bundle` — Flex applies the contrib recipe because §2 enabled it.
2. Verify `config/bundles.php` now has both `FastfonyIdentityBundle` and `StofDoctrineExtensionsBundle`.
3. Verify `config/packages/security.yaml` now has `fastfony_identity_user_provider` + the `fastfony_identity` firewall. If not, apply the reference manually ([identity-bundle.md §4](identity-bundle.md)).
4. Create `config/routes/fastfony_identity.yaml` if Flex didn't.
5. Set `config/packages/mailer.yaml` envelope sender.
6. Run migrations: `symfony console doctrine:migrations:diff` then `migrate`.

Commit:

```bash
git add .
git commit -m "feat: integrate fastfony/identity-bundle + first migration"
```

## 7. First user + smoke test

```bash
symfony console fastfony:user:create      # interactive: email, password
symfony server:start -d                    # https URL printed
curl -sk https://127.0.0.1:<port>/login -o /dev/null -w "%{http_code}\n"   # expect 200
curl -sk https://127.0.0.1:<port>/secure-area/ -o /dev/null -w "%{http_code} → %{redirect_url}\n"   # expect 302 to /login
```

If the user wants public signup exposed, enable it:

```yaml
# config/packages/fastfony_identity.yaml
fastfony_identity:
    registration:
        enabled: true
```

(Registration is `false` by default — without this the `/register` route returns 404.)

Commit:

```bash
git add .
git commit -m "feat: enable public registration"
```

## 8. (Optional) Developer tooling

Only if the user asks. Mirror the curated set below as needed.

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

Already included in `--webapp`. Run `php bin/phpunit` (or `symfony console phpunit` if you installed the bridge).

Commit:

```bash
git add .
git commit -m "chore: add PHPStan + PHP-CS-Fixer"
```

## 9. (Optional) CI

Match the PHP version to the chosen Symfony release:

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
        with:
          php-version: '8.2'    # ← 8.2 for Symfony 7.4, 8.4 for Symfony 8.0
          coverage: none
      - run: composer install --prefer-dist --no-progress
      - run: vendor/bin/phpstan analyse --no-progress
      - run: php bin/console doctrine:migrations:migrate --no-interaction --env=test
      - run: php bin/phpunit
        env:
          DATABASE_URL: postgresql://app:app@127.0.0.1:5432/app_test
```

## 10. Next bundles

As Fastfony grows, add new bundles the same way: `composer require fastfony/<name>-bundle` (contrib already enabled per §2), then follow the bundle's reference file in this skill. Commit each bundle separately.

## Milestone checklist

- [ ] PHP version matches chosen Symfony release
- [ ] Symfony skeleton installed, contrib enabled, initial commit
- [ ] Docker services up (DB + mailer), `symfony console` works
- [ ] Tailwind wired via AssetMapper, `tailwind:build` succeeds
- [ ] `identity-bundle` installed, config verified, migration runs
- [ ] First user created, `/login` 200, `/secure-area/` 302
- [ ] (Optional) Dev tooling + CI
