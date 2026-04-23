# Troubleshooting

Common issues when integrating Fastfony bundles, with the diagnostic command and fix.

## Tables already exist in a fresh project / `doctrine:migrations:diff` says "No changes detected"

**Cause**: Another Postgres container on the host is bound to the default port **5432**, intercepting the project's connection. The webapp recipe's `compose.override.yaml` maps `ports: "5432"` (dynamic high port) to avoid collision, but the project's `.env` hardcodes `DATABASE_URL="postgresql://...@127.0.0.1:5432/app..."`. If `symfony console` doesn't actively inject the Docker-mapped port, raw `php bin/console` / fallback env wins and targets whichever container owns host `5432` — often a leftover from another project like `symfony-database-1`, `some-project-database-1` bound with `ports: "5432:5432"`.

Symptoms that look like "auto-schema":

- `docker compose ps` shows your project DB on a random port (e.g. `32773`).
- `symfony console dbal:run-sql "SELECT ..."` returns tables you never migrated.
- Two different projects on the machine seem to share the same users/migrations.

Diagnose:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep 5432
# Look for any container with "0.0.0.0:5432->5432/tcp" — that one is intercepting.
```

Fix (pick one):

1. **Stop the colliding container** while working on this project:
   ```bash
   docker stop <colliding-container-name>
   ```
2. **Or override `DATABASE_URL` locally** to target your project's actual port (from `docker compose ps`):
   ```bash
   # .env.local
   DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:<actual-port>/app?serverVersion=16&charset=utf8"
   ```
3. **Or** keep using `symfony console` exclusively and verify it's injecting the right URL:
   ```bash
   symfony var:export | grep DATABASE_URL
   ```
   If empty, Symfony CLI isn't injecting — fall back to (2).

Before declaring "auto-schema magic", always check the `user_count` in the DB you *expect* vs the DB you're actually hitting:

```bash
docker exec <your-project-db-container> psql -U app -d app -c 'SELECT COUNT(*) FROM "user";'
```

## Contrib recipe ignored / manual config needed

**Cause**: At install time, `composer require fastfony/identity-bundle` (or `stof/doctrine-extensions-bundle`) printed:

```
IGNORING  fastfony/identity-bundle (>=1.0): From github.com/symfony/recipes-contrib:main
```

The recipe is then marked "installed" in `symfony.lock`, but the `add-lines` operations that inject provider/firewall/access_control into `security.yaml` and register the bundle in `config/bundles.php` were **not** applied. `composer recipes:install --force` replays `copy-from-recipe` but not `add-lines`, so this is not recoverable automatically.

**Prevention** (do this before the first `composer require`):

```bash
composer config extra.symfony.allow-contrib true
```

**Recovery** (if you got bit already):

1. Add the bundles to `config/bundles.php` manually:

   ```php
   Stof\DoctrineExtensionsBundle\StofDoctrineExtensionsBundle::class => ['all' => true],
   Fastfony\IdentityBundle\FastfonyIdentityBundle::class => ['all' => true],
   ```

2. Create `config/packages/stof_doctrine_extensions.yaml` — see [identity-bundle.md §4.1](identity-bundle.md).
3. Rewrite `config/packages/security.yaml` with the provider + firewall + access_control — see [identity-bundle.md §4.2](identity-bundle.md).
4. Create `config/routes/fastfony_identity.yaml` — see [identity-bundle.md §4.3](identity-bundle.md).
5. `symfony console cache:clear` then verify: `symfony console debug:router | grep -iE 'login|register|password|secure-area'` should list the routes.
6. (Still enable `allow-contrib` for future contrib bundles.)

## `Class "Fastfony\IdentityBundle\FastfonyIdentityBundle" does not exist`

**Cause**: Bundle isn't autoloaded, or the Flex recipe didn't register it in `config/bundles.php` (see [Contrib recipe ignored](#contrib-recipe-ignored--manual-config-needed) above — the most common cause).

```bash
composer show fastfony/identity-bundle
grep -n FastfonyIdentityBundle config/bundles.php
composer dump-autoload
```

If the package is installed but bundles.php doesn't mention it, apply the recovery steps above.

## `The "stof_doctrine_extensions" bundle is not registered`

Same root cause as above — the contrib recipe didn't run. Follow [Contrib recipe ignored](#contrib-recipe-ignored--manual-config-needed).

## Login link / password reset email never arrives

Checklist:

1. **`MAILER_DSN` configured?** `php bin/console debug:config framework mailer` → should not show `null://null`.
2. **Async messenger consumer running?** If `framework.messenger.transports.async` routes mailer messages to a queue, you need `php bin/console messenger:consume async` alive. During local dev, either run the consumer or make mail synchronous by commenting out the `send_message_email_async` routing.
3. **Sender domain?** `config/packages/mailer.yaml` envelope `sender:` must be a real address your DSN allows. Many providers (SES, Mailgun) reject unverified senders silently.
4. **Rate limit hit?** `fastfony_identity.login_link.limit_max_ask_by_minute` defaults to 3. Wait a minute or raise the limit.

Inspect logs: `tail -f var/log/dev.log | grep -i mail`.

## `There is no firewall configured for the request`

**Cause**: Security firewall for `fastfony_identity` missing, or route outside the configured firewall pattern.

```bash
php bin/console debug:config security
php bin/console debug:router | grep -i fastfony
```

Ensure `security.firewalls.fastfony_identity` exists (see [identity-bundle.md §4.2](identity-bundle.md)). The default firewall catches all non-dev routes by default (no `pattern:`), so it should be last after the `dev` firewall.

## `/register` returns 404

**Cause**: Registration is disabled. Two sub-cases:

1. **No `config/packages/fastfony_identity.yaml` at all** → the bundle's internal default (`enabled: false`) applies. Happens when Flex didn't run or contrib wasn't enabled (see [Contrib recipe ignored](#contrib-recipe-ignored--manual-config-needed)).
2. **The yaml exists but was manually edited to `enabled: false`** → change it back to `true`.

Fix in `config/packages/fastfony_identity.yaml`:

```yaml
fastfony_identity:
    registration:
        enabled: true
```

Note: the Flex recipe-generated yaml already sets `enabled: true`, so a normal Flex install exposes `/register` with no extra step. See [identity-bundle.md — Registration default](identity-bundle.md#registration-default-bundle-internal-vs-flex-recipe).

## `doctrine:migrations:diff` produces no changes

**Cause**: Schema already matches the DB, or bundle entities aren't detected.

```bash
php bin/console doctrine:mapping:info
```

Should list `Fastfony\IdentityBundle\Entity\Identity\User`, `Role`, `Group`, and `RequestPassword`. If not:

1. `composer dump-autoload`
2. `php bin/console cache:clear`
3. Check that the bundle is registered in `config/bundles.php`.

## `Cannot autowire service ... Fastfony\IdentityBundle\Manager\UserManager`

**Cause**: Service wiring isn't picked up — usually a Flex misfire or missing `services.yaml` import.

The bundle auto-registers its services via `config/services.yaml`. If it's not loading:

```bash
php bin/console debug:container | grep -i fastfony
```

Should list the managers. If not, `cache:clear` and verify `vendor/fastfony/identity-bundle/config/services.yaml` exists.

## Overridden template is ignored

**Cause**: Wrong directory or wrong extends path.

- Directory must be exactly `templates/bundles/FastfonyIdentityBundle/` (case-sensitive, bundle class basename without `.php`).
- To extend the original from your override, use `{% extends '@!FastfonyIdentity/form_login.html.twig' %}` (note the `!`).

Debug:

```bash
php bin/console debug:twig --filter=path
```

## `tailwind:build` fails with `signal 9` / `Process has been signaled with signal "9"`

**Cause**: The Tailwind standalone binary downloaded by `symfony console tailwind:init` is **truncated** (partial download). Neither the Symfony CLI nor `tailwind:init` detect this — the file lands at `var/tailwind/v<version>/tailwindcss-<os>-<arch>` with a short size, and every invocation gets SIGKILL at exec time. The direct binary exits 137 (`128 + 9 = SIGKILL`), often silently on `--help`.

Symptoms:

```
 [ERROR] Tailwind CSS build failed: see output above.
 # or
In Process.php line 488:
  The process has been signaled with signal "9".
```

Diagnose — compare the binary size to the full release size:

```bash
ls -l var/tailwind/v*/tailwindcss-*
# Full Tailwind v4.1.11 on macos-arm64 is ~75 MB.
# If you see ~51 MB or any size < 70 MB, the download was truncated.
```

Fix (pick one):

1. **Delete and re-init** — lets `tailwind:init` re-download from scratch:
   ```bash
   rm -rf var/tailwind/
   symfony console tailwind:init
   symfony console tailwind:build
   ```
2. **Copy from a known-good project** if you have one on the same machine + same OS/arch:
   ```bash
   cp /path/to/working-project/var/tailwind/v4.1.11/tailwindcss-macos-arm64 var/tailwind/v4.1.11/
   chmod +x var/tailwind/v4.1.11/tailwindcss-macos-arm64
   symfony console tailwind:build
   ```
3. **Download manually** from the Tailwind releases page and drop it in:
   ```bash
   VERSION=4.1.11
   curl -L -o var/tailwind/v${VERSION}/tailwindcss-macos-arm64 \
     "https://github.com/tailwindlabs/tailwindcss/releases/download/v${VERSION}/tailwindcss-macos-arm64"
   chmod +x var/tailwind/v${VERSION}/tailwindcss-macos-arm64
   ```

After the fix, `tailwind:build` prints `Done in XXms` and the CSS compiles.

Unrelated note: on macOS, running `--help` on a completely broken binary may exit 0 with no output (Gatekeeper / dyld quirk), while execing it via `Process` triggers SIGKILL. Don't trust a silent `--help` as proof the binary is intact — check the file size.

## `Unknown "tailwind_stylesheet" function in "base.html.twig"`

**Cause**: Following outdated tailwind-bundle documentation. The `tailwind_stylesheet()` Twig helper was removed in `symfonycasts/tailwind-bundle ≥ 0.11` (which is what you get today with Tailwind v4).

**Fix**: Do nothing special in the template. The integration flows through AssetMapper:

1. `assets/app.js` already contains `import './styles/app.css';` (added by the webapp recipe).
2. `assets/styles/app.css` should start with `@import "tailwindcss";` (added by `symfony console tailwind:init`).
3. `base.html.twig` already has `{{ importmap('app') }}`, which emits the compiled CSS link automatically.

Just run `symfony console tailwind:build` (one-shot) or `symfony console tailwind:build --watch` (dev) to (re)compile the CSS. Remove any `{{ tailwind_stylesheet() }}` call if you added one.

## Login form styles look broken

The bundle ships HTML templates with inline `<style>` blocks so they render acceptably out of the box. Once Tailwind is wired (above), override the templates in `templates/bundles/FastfonyIdentityBundle/` to apply your design system:

- `form_login.html.twig`, `register.html.twig`, `request_login_link.html.twig`, `forgot_password.html.twig`, `reset_password.html.twig`
- `emails/` subdirectory for transactional email templates

Import custom CSS via `importmap:require`, or just add Tailwind classes to the overrides. Do not introduce Webpack Encore or a JS bundler to solve styling.

## Can I use `fastfony/fastfony` (the monolithic starter-kit) alongside these bundles?

No — pick one model. The bundle model is what we recommend now. If the user insists on the legacy kit, warn them it's intentionally being deprecated as a starter option, then proceed only on their explicit confirmation.
