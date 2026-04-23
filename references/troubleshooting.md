# Troubleshooting

Common issues when integrating Fastfony bundles, with the diagnostic command and fix.

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

**Cause**: Registration is disabled by default.

Fix in `config/packages/fastfony_identity.yaml`:

```yaml
fastfony_identity:
    registration:
        enabled: true
```

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
