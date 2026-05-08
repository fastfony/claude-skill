# Fastfony conventions

Apply these when writing code that will live alongside Fastfony bundles, and especially when creating a new bundle to publish under `fastfony/*`.

## PHP & Symfony baseline

- **PHP**: minimum depends on the **Symfony** release.
  - A Fastfony **bundle** keeps PHP ≥ 8.1 (to maximize downstream compat) and declares `"symfony/*": "^7.4|^8.0"`.
  - An **application** built with this skill follows the chosen Symfony version:
    - Symfony **7.4 LTS** → PHP ≥ **8.2**
    - Symfony **8.0** → PHP ≥ **8.4**
  - Never state "PHP 8.4 required" in a 7.4 LTS starter-kit.
- **Symfony**: default to 7.4 LTS for new starter-kits unless the user asks for 8.0.
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

## Frontend / assets

**AssetMapper only.** Starter-kits and bundles generated or extended via this skill must use Symfony AssetMapper as the sole asset handler. No Webpack Encore, no Vite, no bundlers, no Node build step. This is a hard rule — it differentiates current Fastfony kits from the legacy `fastfony/fastfony` monolith (which used Encore + Vue).

What's allowed:

- **Tailwind CSS** via `symfonycasts/tailwind-bundle` (Tailwind v4 integration). The helper `tailwind_stylesheet()` was removed in bundle v0.11+ — do **not** call it. The integration flows entirely through AssetMapper: `assets/app.js` imports `./styles/app.css`, and `{{ importmap('app') }}` in `base.html.twig` emits the link tag. Dev: `symfony console tailwind:build --watch` in a second terminal. Prod: `symfony console tailwind:build` + `symfony console asset-map:compile`.
- **Bootstrap** via pure importmap (no dedicated bundle needed):
  ```bash
  symfony console importmap:require bootstrap
  ```
  This adds three entries to `importmap.php` in one go: the JS package, `@popperjs/core` (dependency for tooltips/dropdowns), and `bootstrap/dist/css/bootstrap.min.css` (with `type: 'css'`). Then in `assets/app.js`:
  ```js
  import 'bootstrap/dist/css/bootstrap.min.css';
  import './styles/app.css';   // your overrides on top
  import 'bootstrap';           // enables data-bs-* components
  ```
  No build step, no `package.json`. `{{ importmap('app') }}` emits both the CSS link and the JS module tag. Prod: `symfony console asset-map:compile`.
- **Plain CSS**: imported through the importmap like any asset.
- **JS interactivity**: **Stimulus** controllers (`symfony/stimulus-bundle`) and **Turbo** (`symfony/ux-turbo`) — both wire cleanly into AssetMapper, and are complementary to Tailwind or Bootstrap.
- **Third-party JS**: add via `symfony console importmap:require <package>` (uses JSPM / jsDelivr CDN resolution). Avoid anything that requires a build step (JSX, TSX, SCSS preprocessing beyond Tailwind).

Tailwind and Bootstrap can coexist (not recommended, but possible). Pick one per starter-kit based on the user's preference. Default to Tailwind if they don't care — Fastfony's own bundle templates are lighter-weight there.

What's NOT allowed:

- Webpack Encore (`@symfony/webpack-encore`), Vite, Parcel, Rollup, or any bundler.
- `package.json` for runtime deps. A `package.json` may exist only for Tailwind CLI if the user opts into it, but prefer the Symfony Tailwind bundle which handles this transparently.
- Vue, React, Svelte, Angular — no SPA frameworks as build-step deps. If a feature truly needs client-side reactivity beyond Stimulus/Turbo, open an issue rather than introducing a bundler.

Migrating a user's legacy Encore setup to AssetMapper is in scope for this skill; installing Encore is not.

## Code quality

Minimum for a bundle to be publishable under `fastfony/*`:

- **PHPStan level 6+** (aim for 8 on new bundles).
- **PHPUnit** with unit + functional coverage for every public Manager/Controller.
- **PHP-CS-Fixer** with `@Symfony` + `@Symfony:risky` + `declare_strict_types`.
- **No TODO/FIXME** in main — open an issue instead.

## Licensing

- **Everything Fastfony publishes is MIT**: bundles (e.g. `identity-bundle`), packs (e.g. `quality-pack`), and the Claude Code skill itself. Easy adoption is the whole point.
- **A starter-kit built by a user from Fastfony bundles**: whatever license the user chooses.
- The legacy `fastfony/fastfony` monolithic kit was Apache-2.0 + Commons-Clause (commercial use paid). It's deprecated as a starting point — don't confuse it with the current MIT-only model.

## Naming

- Bundle packages: `fastfony/{noun}-bundle` (singular noun): `identity-bundle`, `blog-bundle`, `billing-bundle`.
- Extension alias / YAML file: `fastfony_{noun}` (singular, snake_case).
- Main class: `Fastfony{Noun}Bundle`.
- Route name prefix: `fastfony_{noun}_*` to avoid collisions with app routes.
