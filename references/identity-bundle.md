# `fastfony/identity-bundle` — integration guide

Source: [github.com/fastfony/identity-bundle](https://github.com/fastfony/identity-bundle) · License: MIT · Namespace: `Fastfony\IdentityBundle\`

Provides: form login, magic login link, registration with optional email verification, password reset, user/role/group entities, login throttling, last-login tracking.

## 1. Prerequisites

- PHP ≥ 8.1 (recommend 8.2+ for a modern starter-kit)
- Symfony 7.4 LTS or 8.0
- Doctrine ORM (bundle adds it as dep if missing)
- A working `MAILER_DSN` (required for login link, registration confirmation, password reset)

Check with:

```bash
php -v
composer show symfony/framework-bundle
composer show doctrine/orm 2>/dev/null || echo "Doctrine not installed yet"
```

## 2. Install

```bash
composer require fastfony/identity-bundle
```

This pulls in transitive deps automatically:

- `stof/doctrine-extensions-bundle` (Timestampable)
- `symfony/mailer`, `symfony/notifier`, `symfony/twig-bundle`, `twig/extra-bundle`, `twig/cssinliner-extra`, `twig/inky-extra` (emails)
- `symfony/rate-limiter` (login throttling)
- `symfony/translation`, `symfony/uid`, `symfony/form`

If Symfony Flex is active, the recipe registers the bundle and creates config files automatically — **skip to section 5**. Otherwise continue with manual configuration.

## 3. Register the bundle (no Flex)

Edit `config/bundles.php`:

```php
return [
    // ...
    Fastfony\IdentityBundle\FastfonyIdentityBundle::class => ['all' => true],
    Stof\DoctrineExtensionsBundle\StofDoctrineExtensionsBundle::class => ['all' => true],
];
```

## 4. Manual configuration (no Flex)

### 4.1 Doctrine Extensions

Create `config/packages/stof_doctrine_extensions.yaml`:

```yaml
stof_doctrine_extensions:
    default_locale: en_US
    orm:
        default:
            timestampable: true
```

### 4.2 Security firewall

Edit `config/packages/security.yaml` — add the provider, firewall, and access control. Preserve any existing firewalls (the `dev` firewall must stay first):

```yaml
security:
    # ... existing password_hashers, etc.

    providers:
        fastfony_identity_user_provider:
            entity:
                class: Fastfony\IdentityBundle\Entity\Identity\User
                property: email

    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false

        fastfony_identity:
            lazy: true
            provider: fastfony_identity_user_provider
            user_checker: Fastfony\IdentityBundle\Security\UserChecker
            form_login:
                login_path: form_login
                check_path: form_login
                enable_csrf: true
                csrf_token_id: login
                form_only: true
                default_target_path: fastfony_identity_secure_area
            login_link:
                check_route: login_check
                signature_properties: [id, email]
                max_uses: 3
                default_target_path: fastfony_identity_secure_area
            entry_point: Fastfony\IdentityBundle\Security\CustomEntryPoint
            remember_me:
                always_remember_me: true
                signature_properties: [id, email, password]
            switch_user: true
            login_throttling:
                max_attempts: 3
            logout:
                path: /logout
                clear_site_data:
                    - cookies
                    - storage

    access_control:
        - { path: ^/secure-area/, roles: ROLE_USER }
```

### 4.3 Routing

Create `config/routes/fastfony_identity.yaml`:

```yaml
fastfony_identity:
    resource: "@FastfonyIdentityBundle/config/routes/all.yaml"
```

This exposes (among others):

| Route name | Path | Purpose |
|---|---|---|
| `form_login` | `/login` (default) | Form login page |
| `login_check` | `/login_check` | Form submission target |
| `fastfony_identity_register` | `/register` | Registration form |
| `fastfony_identity_request_password` | `/request-password` | Password reset request |
| `fastfony_identity_secure_area` | `/secure-area/` | Default authenticated landing |

Confirm with `php bin/console debug:router | grep -i fastfony` after setup.

### 4.4 Mailer envelope

Edit `config/packages/mailer.yaml`:

```yaml
framework:
    mailer:
        dsn: '%env(MAILER_DSN)%'
        envelope:
            sender: 'noreply@your-domain.tld'
```

Set `MAILER_DSN` in `.env.local` (or via your infra). During local dev, `null://null` disables sending — but login link and password reset won't work. Use something like Mailpit / MailHog / Mailtrap for local testing.

## 5. Database schema

### Option A — migrations (recommended for any real project)

```bash
php bin/console doctrine:migrations:diff
php bin/console doctrine:migrations:migrate
```

### Option B — direct schema update (dev only, no existing data)

```bash
php bin/console doctrine:schema:update --force
```

**Never use Option B on a database with production data.**

Tables created: `user`, `role`, `group`, `user_role`, `user_group`, `group_role`, `request_password`.

## 6. Create the first user

### Option A — via command (works offline, no mailer needed)

```bash
php bin/console fastfony:user:create
```

Interactive prompt for email, password, roles.

### Option B — via browser

Visit `/register`. Requires `MAILER_DSN` to be functional (and sync mailer, or an active `messenger:consume` worker) so the activation / login-link email actually arrives.

## 7. Verify

```bash
php bin/console about
php bin/console debug:router | grep -i fastfony
php bin/console debug:config security
```

Start the server and navigate to `/login`. A login should redirect to `/secure-area/`.

---

## Bundle configuration reference

Full config tree with defaults (create `config/packages/fastfony_identity.yaml` to override):

```yaml
fastfony_identity:
    user:
        require_email_verification: false        # true → user must click email link before login
    role:
        class: Fastfony\IdentityBundle\Entity\Identity\Role
        default_role: ROLE_USER                  # assigned to every new user
    group:
        class: Fastfony\IdentityBundle\Entity\Identity\Group
    registration:
        enabled: false                           # ⚠️ disabled by default — enable explicitly
    login:
        default_method: form_login               # or "login_link"
    login_link:
        enabled: true
        email_subject: 'Your login link'
        limit_max_ask_by_minute: 3
    request_password:
        email_subject: 'Reset your password'
        email_content: 'Click on the button below to reset your password.'
        email_action_text: 'Reset my password'
        lifetime: 900                            # seconds (15 min)
        redirect_route: fastfony_identity_secure_area
```

**Registration is disabled by default.** To expose `/register`, set `registration.enabled: true`.

---

## Customization

### Override templates

Place overrides in `templates/bundles/FastfonyIdentityBundle/`. You can either fully replace or extend the original:

```
templates/bundles/FastfonyIdentityBundle/
├── form_login.html.twig
├── register.html.twig
├── request_password.html.twig
└── emails/
    ├── login_link.html.twig
    ├── registration_confirmation.html.twig
    └── request_password.html.twig
```

Extend a default and change one block:

```twig
{# templates/bundles/FastfonyIdentityBundle/form_login.html.twig #}
{% extends '@!FastfonyIdentity/form_login.html.twig' %}

{% block title %}Sign in to {{ parent() }}{% endblock %}
```

The `@!` prefix references the original bundle template (Symfony's template override convention).

### Custom User / Role / Group entities

Extend the bundle entity and point config at yours. Example for User:

```php
// src/Entity/User.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Fastfony\IdentityBundle\Entity\Identity\User as BaseUser;

#[ORM\Entity]
#[ORM\Table(name: 'user')]
class User extends BaseUser
{
    #[ORM\Column(length: 120, nullable: true)]
    private ?string $displayName = null;

    public function getDisplayName(): ?string { return $this->displayName; }
    public function setDisplayName(?string $v): self { $this->displayName = $v; return $this; }
}
```

Update `config/packages/security.yaml` provider `class:` to `App\Entity\User`.

For Role/Group, set `fastfony_identity.role.class` / `fastfony_identity.group.class` to your extended class.

### Managers (preferred over direct repository access)

Inject the appropriate manager in services/controllers:

- `Fastfony\IdentityBundle\Manager\UserManager` — `create()`, `updatePassword()`, `enable()`/`disable()`, `findByEmail()`, `updateLastLogin()`
- `Fastfony\IdentityBundle\Manager\RoleManager` — CRUD + `getAll()` (ordered by name)
- `Fastfony\IdentityBundle\Manager\GroupManager` — CRUD + `getAll()`, `findByName()`

All managers flush on persist. If you need batched writes, open an issue upstream rather than bypassing the manager.

### Events

`Symfony\Component\Security\Http\Event\LoginSuccessEvent` → the bundle's subscriber updates `user.lastLogin` automatically. Subscribe to the same event from app code if you need additional behavior (audit log, metric, etc.) — the bundle's subscriber uses default priority.

---

## Key entity relationships

```
User ─ * ─── * ─ Role
 │
 * ─── * ─ Group ─── * ─── * ─ Role
```

- User ↔ Role: many-to-many
- User ↔ Group: many-to-many
- Group ↔ Role: many-to-many (users inherit group roles)
- All three entities use Timestampable (`createdAt`, `updatedAt`)

`RequestPassword` is a separate entity scoped by user with a UUIDv7 token and an expiry.
