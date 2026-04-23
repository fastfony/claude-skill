# Troubleshooting

Common issues when integrating Fastfony bundles, with the diagnostic command and fix.

## `Class "Fastfony\IdentityBundle\FastfonyIdentityBundle" does not exist`

**Cause**: Bundle isn't autoloaded or the package didn't install.

```bash
composer show fastfony/identity-bundle
composer dump-autoload
```

If missing, reinstall: `composer require fastfony/identity-bundle`.

## `The "stof_doctrine_extensions" bundle is not registered`

**Cause**: Flex recipe didn't run (common when `--no-interaction` blocks prompts) and `StofDoctrineExtensionsBundle` isn't in `config/bundles.php`.

Fix — add it manually:

```php
// config/bundles.php
Stof\DoctrineExtensionsBundle\StofDoctrineExtensionsBundle::class => ['all' => true],
```

And ensure `config/packages/stof_doctrine_extensions.yaml` exists (see [identity-bundle.md §4.1](identity-bundle.md)).

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

## Login form styles look broken

The bundle ships unstyled HTML templates. Apply your own CSS by overriding the template in `templates/bundles/FastfonyIdentityBundle/`. Fastfony starter-kits use AssetMapper for CSS delivery — import your stylesheet via `importmap:require` and reference it in `base.html.twig`. Do not introduce Webpack Encore or a JS bundler to solve styling.

## Can I use `fastfony/fastfony` (the monolithic starter-kit) alongside these bundles?

No — pick one model. The bundle model is what we recommend now. If the user insists on the legacy kit, warn them it's intentionally being deprecated as a starter option, then proceed only on their explicit confirmation.
