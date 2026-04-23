# Fastfony conventions

Apply these when writing code that will live alongside Fastfony bundles, and especially when creating a new bundle to publish under `fastfony/*`.

## PHP & Symfony baseline

- **PHP** ≥ 8.1 for a bundle (compatibility), ≥ 8.2 for an application. Prefer 8.3+ for new starter-kits.
- **Symfony** 7.4 LTS or 8.0. Bundles declare `"symfony/*": "^7.4|^8.0"`.
- **`declare(strict_types=1);`** at the top of every PHP file.
- **Final classes** by default. Only drop `final` when a class is explicitly designed for extension (e.g. `BaseUser` in a bundle). Document the extension contract in that case.
- **Constructor property promotion** everywhere.
- **Named arguments** at call sites when they improve readability (≥ 3 args, or when the argument meaning isn't obvious).

## Namespaces

- Applications: `App\...` (Symfony default).
- Bundles: `Fastfony\{Name}Bundle\...`. The main bundle class lives directly under the bundle namespace: `Fastfony\{Name}Bundle\Fastfony{Name}Bundle`.
- Bundle PSR-4 in `composer.json`:
  ```json
  "autoload": {
      "psr-4": {
          "Fastfony\\XxxBundle\\": "src/"
      }
  },
  "autoload-dev": {
      "psr-4": {
          "Fastfony\\XxxBundle\\Tests\\": "tests/"
      }
  }
  ```

## Directory layout

### Application

```
src/
├── Command/
├── Controller/
├── Entity/
├── EventSubscriber/
├── Form/
├── Manager/            # ← CRUD lives here (not in repositories)
├── Repository/         # ← query methods only
├── Security/
└── Twig/
```

### Bundle

```
src/
├── Command/
├── Controller/
├── DependencyInjection/    # optional, only if not using AbstractBundle
├── Entity/
│   └── {Feature}/          # group related entities in a subfolder
├── EventSubscriber/
├── Form/
├── Manager/
├── Notifier/
├── Repository/
├── Security/
└── Fastfony{Name}Bundle.php
config/
├── routes/
│   └── all.yaml
└── services.yaml
docs/
├── configuration.md
├── customization.md
├── index.md
└── miscellaneous.md
templates/                  # bundle's own templates
translations/
tests/
```

## Entity conventions

- Every domain entity uses Stof's **Timestampable** (`#[Gedmo\Timestampable]` or the trait shipped by stof) — expose `createdAt` and `updatedAt`.
- Prefer **UUIDv7** (`Symfony\Component\Uid\Uuid::v7()`) for public-facing identifiers (password reset tokens, invitation codes, etc.). Auto-increment integers are fine for internal PKs.
- Use the **`#[ORM\Table(name: '...')]`** attribute with an explicit lowercase snake_case table name to avoid surprises when DBs differ in case sensitivity.
- Collections: type hints `Collection<int, EntityType>` and initialize in the constructor.

## Manager pattern (important)

Fastfony separates read and write concerns:

- **Repository**: only query methods (`findActive()`, `findByEmailDomain()`, etc.). No persistence.
- **Manager**: the write API (`create()`, `update*()`, `enable()`, `delete()`). Always flushes unless there's a documented reason not to.

Application code depends on the manager. Controllers/commands **must not** call `$entityManager->persist()` or `->flush()` directly on domain entities that have a manager.

Example:

```php
final class UserManager
{
    public function __construct(
        private readonly EntityManagerInterface $em,
        private readonly UserPasswordHasherInterface $hasher,
    ) {}

    public function create(string $email, string $plainPassword, array $roles = ['ROLE_USER']): User
    {
        $user = (new User())
            ->setEmail($email)
            ->setRoles($roles);

        $user->setPassword($this->hasher->hashPassword($user, $plainPassword));

        $this->em->persist($user);
        $this->em->flush();

        return $user;
    }
}
```

## Configuration conventions

- One YAML file per bundle in `config/packages/`, named after the bundle extension alias: `fastfony_identity.yaml`, `fastfony_blog.yaml`, etc.
- Bundle routes imported into `config/routes/{alias}.yaml` with a single `resource:` line.
- Override templates under `templates/bundles/{BundleName}/`.
- Bundle main class extends `Symfony\Component\HttpKernel\Bundle\AbstractBundle` and defines `configure()` + `loadExtension()` (no separate `DependencyInjection/` directory needed for simple cases).

## Code quality

Minimum for a bundle to be publishable under `fastfony/*`:

- **PHPStan level 6+** (aim for 8 on new bundles).
- **PHPUnit** with unit + functional coverage for every public Manager/Controller.
- **PHP-CS-Fixer** with `@Symfony` + `@Symfony:risky` + `declare_strict_types`.
- **No TODO/FIXME** in main — open an issue instead.

## Licensing

- **Bundles published under `fastfony/`**: MIT (like `identity-bundle`). Easy adoption is the whole point.
- **A starter-kit built by a user from Fastfony bundles**: whatever license the user chooses.
- The legacy `fastfony/fastfony` monolithic kit is Apache-2.0 + Commons-Clause (commercial use paid). Do not confuse the two models.

## Naming

- Bundle packages: `fastfony/{noun}-bundle` (singular noun): `identity-bundle`, `blog-bundle`, `billing-bundle`.
- Extension alias / YAML file: `fastfony_{noun}` (singular, snake_case).
- Main class: `Fastfony{Noun}Bundle`.
- Route name prefix: `fastfony_{noun}_*` to avoid collisions with app routes.
