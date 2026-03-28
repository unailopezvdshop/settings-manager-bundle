# CLAUDE.md — Helis Settings Manager Bundle

This file provides AI assistants with the context needed to understand and contribute to this codebase effectively.

---

## Project Overview

`helis/settings-manager-bundle` is a **Symfony Bundle** for centralized application settings management. It supports multiple backend providers (database, Redis, cookies, AWS SSM, in-memory), a typed domain model, a web UI, a CLI, and an event-driven extension system.

- **PHP:** >=7.1.3
- **Symfony:** ^3.4 | ^4.3
- **Namespace root:** `Helis\SettingsManagerBundle\`
- **Bundle entry point:** `src/HelisSettingsManagerBundle.php`

---

## Repository Structure

```
settings-manager-bundle/
├── src/
│   ├── Command/                  # CLI commands (debug, warm-up, key generation)
│   ├── Controller/               # HTTP controllers for settings UI
│   ├── DataCollector/            # Symfony Profiler integration
│   ├── DependencyInjection/      # Bundle extension + 3 compiler passes
│   │   └── Compiler/             # ProviderFactoryPass, ProviderPass, SettingsAwarePass
│   ├── Enqueue/                  # Message queue integration
│   ├── Event/                    # Event classes (Get, Change, Menu)
│   ├── Exception/                # Custom exception hierarchy
│   ├── Form/Type/                # Symfony Form types (Setting, Yaml, Domain)
│   ├── Menu/                     # KNPMenuBundle integration
│   ├── Model/                    # Domain models: SettingModel, DomainModel, TagModel, Type enum
│   ├── Monolog/                  # Logging processor
│   ├── Provider/                 # 14 provider implementations + interfaces + traits
│   │   └── Factory/              # ProviderFactoryInterface + SimpleSettingsProviderFactory
│   ├── Resources/
│   │   └── config/               # services.yml, routing.yml, Doctrine mappings, translations
│   ├── Serializer/Normalizer/    # SettingModel, DomainModel, TagModel normalizers
│   ├── Settings/                 # Core business logic (SettingsManager, SettingsRouter, SettingsStore)
│   │   ├── Switchable/           # Interfaces + trait for bool-setting injection
│   │   └── Traits/               # SettingsRouterAwareTrait, DomainNameExtractTrait
│   ├── Subscriber/               # Event subscribers for controllers and commands
│   ├── Twig/                     # SettingsExtension (setting_get Twig function)
│   └── Validator/Constraints/    # UniqueSetting, SettingType validators
├── tests/
│   ├── src/
│   │   ├── Unit/                 # Unit tests per module
│   │   └── Functional/           # Integration tests using test kernel
│   ├── app/config/               # Test kernel configuration YAML files
│   └── bundle/                   # AppBundle test fixtures, entities, commands
├── benchmarks/                   # PHPBench benchmarks
├── composer.json
├── phpunit.xml.dist
├── phpbench.json
└── .php_cs                       # PHP CS Fixer config
```

---

## Core Architecture

### Settings Flow

```
SettingsRouter (public API)
    └── SettingsManager (orchestrator)
            ├── SettingsStore (in-memory cache)
            └── [Provider1, Provider2, ...] (priority-ordered)
```

- **`SettingsRouter`** (`src/Settings/SettingsRouter.php`) is the public-facing API. Use `get()`, `getString()`, `getBool()`, `getInt()`, `getFloat()`, `getArray()` to retrieve values.
- **`SettingsManager`** (`src/Settings/SettingsManager.php`) orchestrates providers, handles save/delete/copy, and dispatches events.
- **`SettingsStore`** (`src/Settings/SettingsStore.php`) provides an in-memory cache layer so providers are not hit on every call.

### Domains

Every setting belongs to a **domain** (`DomainModel`). A domain has:
- `name` (string identifier)
- `priority` (int — higher wins in provider merging)
- `enabled` (bool)
- `readOnly` (bool — prevents writes)

The `default` domain always exists.

### Setting Types (`src/Model/Type.php`)

An enum backed by `myclabs/php-enum`:

| Value | Constant |
|-------|----------|
| `string` | `Type::STRING` |
| `bool` | `Type::BOOL` |
| `int` | `Type::INT` |
| `float` | `Type::FLOAT` |
| `yaml` | `Type::YAML` |
| `choice` | `Type::CHOICE` |

### Providers

All providers implement `SettingsProviderInterface`. The bundle ships 14 implementations:

| Provider | Description |
|----------|-------------|
| `SimpleSettingsProvider` | In-memory, config-defined settings |
| `ReadableSimpleSettingsProvider` | Read-only in-memory |
| `LazyReadableSimpleSettingsProvider` | Lazy-loaded read-only in-memory |
| `DoctrineOrmSettingsProvider` | Persists to database via Doctrine ORM |
| `CookieSettingsProvider` | Symmetric-encrypted cookie storage |
| `AsymmetricCookieSettingsProvider` | Asymmetric-encrypted cookie storage |
| `AwsSsmSettingsProvider` | AWS Systems Manager Parameter Store |
| `DecoratingRedisSettingsProvider` | Redis caching decorator (phpredis) |
| `DecoratingPredisSettingsProvider` | Redis caching decorator (predis) |
| `DecoratingCacheSettingsProvider` | Symfony Cache component decorator |
| `DecoratingInMemorySettingsProvider` | In-memory caching decorator |
| `DecoratingPhpFilesSettingsProvider` | PHP file-based caching decorator |

Decorating providers wrap another provider and add a caching layer — they do not store primary data themselves.

### Events (`src/SettingsManagerEvents.php`)

| Constant | When |
|----------|------|
| `GET_SETTING` | After a setting is retrieved |
| `PRE_EDIT_SETTING` | Before a setting is saved or deleted |
| `SAVE_SETTING` | After a setting is saved |
| `DELETE_SETTING` | After a setting is deleted |
| `CONFIGURE_MENU` | When building the UI menu |

### Compiler Passes (`src/DependencyInjection/Compiler/`)

1. **`ProviderFactoryPass`** — Collects services tagged `settings_manager.provider_factory` and registers them.
2. **`ProviderPass`** — Collects services tagged `settings_manager.provider` (with optional `priority` attribute) and registers them in `SettingsManager`.
3. **`SettingsAwarePass`** — Collects services tagged `settings_manager.setting_aware` and wraps them in a lazy proxy that injects settings via setter methods.

---

## Service Tags (DI)

| Tag | Purpose |
|-----|---------|
| `settings_manager.provider` | Register a provider; optional `priority` attr |
| `settings_manager.provider_factory` | Register a provider factory |
| `settings_manager.setting_aware` | Inject settings into a service via setter; use `method` and `setting` attrs |

---

## Code Conventions

### Required in every PHP file

```php
<?php

declare(strict_types=1);
```

### Strict Typing

- All method parameters must have type hints.
- All methods must have explicit return types (`void`, `?string`, `array`, etc.).
- Use nullable types (`?string`) rather than defaulting to `null` without annotation.

### Style (enforced by `.php_cs` / PHP CS Fixer)

- **Short array syntax:** `[]` not `array()`
- **No Yoda conditions:** `$a === $b`, not `$b === $a`
- **PSR-2 + Symfony** ruleset
- **`no_null_property_initialization`:** Don't initialize properties to `null`

Run the fixer:
```bash
composer phpcs       # check
composer phpcs-fix   # auto-fix
```

### Naming

| Type | Convention | Example |
|------|-----------|---------|
| Class | PascalCase | `SettingsManager` |
| Method | camelCase | `getSettingsByName()` |
| Constant | UPPER_SNAKE_CASE | `DEFAULT_PROVIDER` |
| Property | camelCase | `$settingsStore` |

### Model Setters

Model classes use fluent setters returning `$this`:

```php
$setting = (new SettingModel())
    ->setName('my_setting')
    ->setType(Type::STRING())
    ->setData('value');
```

### PHPDoc

Add `@param`, `@return`, `@throws` docblocks when the signature alone is not self-documenting. Do not add redundant docblocks that simply restate the type hints.

---

## Development Commands

```bash
# Install dependencies
composer install

# Run tests
./vendor/bin/phpunit

# Static analysis
composer phpstan

# Code style check
composer phpcs

# Code style fix
composer phpcs-fix

# Performance benchmarks
./vendor/bin/phpbench run benchmarks/
```

### Test Environment

Tests use a full Symfony kernel defined at `tests/AppKernel.php`. Database tests require a configured database — Doctrine ORM fixtures are loaded via `LiipTestFixturesBundle`.

Required env vars for tests (set in `phpunit.xml.dist`):
- `REDIS_HOST` (default: `localhost`)
- `REDIS_PORT` (default: `6379`)

---

## Adding a New Provider

1. Create a class in `src/Provider/` implementing `SettingsProviderInterface`.
2. Use `WritableProviderTrait` or `ReadOnlyProviderTrait` if appropriate.
3. Register it as a service in `src/Resources/config/services.yml` and tag it with `settings_manager.provider`.
4. Add functional tests under `tests/src/Functional/Provider/`.
5. Document configuration in `README.md`.

For a **decorating** (caching) provider, also implement `ModificationAwareSettingsProviderInterface` to track when the underlying data changes.

---

## Adding a New Setting Type

1. Add the value to `src/Model/Type.php` (extends `MyCLabs\Enum\Enum`).
2. Handle the new type in `SettingsManager` and any serializer/normalizer logic.
3. Add a form field mapping in `src/Form/Type/SettingFormType.php`.
4. Add a validator case in `src/Validator/Constraints/SettingTypeValidator.php`.

---

## Key Files Reference

| File | Role |
|------|------|
| `src/HelisSettingsManagerBundle.php` | Bundle bootstrap; registers Doctrine type and compiler passes |
| `src/Settings/SettingsRouter.php` | **Public API** — use this in application code |
| `src/Settings/SettingsManager.php` | Core orchestrator; do not use directly in app code |
| `src/Model/SettingModel.php` | Domain model for a single setting |
| `src/Model/Type.php` | Enum of valid setting types |
| `src/Provider/SettingsProviderInterface.php` | Contract all providers must implement |
| `src/DependencyInjection/Configuration.php` | Bundle YAML configuration schema |
| `src/Resources/config/services.yml` | All service definitions |
| `src/SettingsManagerEvents.php` | Event name constants |

---

## Testing Patterns

- **Unit tests** mock dependencies; live in `tests/src/Unit/`.
- **Functional tests** boot the full test kernel; live in `tests/src/Functional/`.
- Use `LiipFunctionalTestBundle` for kernel-level setup and fixture loading.
- Provider tests extend a common base class and test the full CRUD contract.

When writing new tests:
- Place them in the correct `Unit/` or `Functional/` subdirectory mirroring the `src/` path.
- Name test classes `<ClassName>Test` and test methods `test<Behaviour>`.
- Use PHPUnit data providers for type or variant coverage.

---

## Git Workflow

- Active development branch: `claude/add-claude-documentation-2ky5S`
- Main stable branch: `master`
- Push with: `git push -u origin <branch-name>`
- Write clear, imperative commit messages (e.g., `Add Redis decorator provider`, `Fix null type coercion in SettingsRouter`).
