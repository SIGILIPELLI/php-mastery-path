# 09 · Working with Composer Packages at Scale

Every PHP project so far has used Composer for autoloading and a handful of
dependencies. At the scale of a real application — dozens of packages,
multiple developers, a CI pipeline — a few more things start to matter:
version constraints, the lock file, splitting `require` from `require-dev`,
and keeping the autoloader itself fast.

## `composer.json` for a real package

```json
{
    "name": "acme/reporting",
    "description": "Internal reporting library",
    "license": "proprietary",
    "require": {
        "php": "^8.2"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0"
    },
    "autoload": {
        "psr-4": { "Acme\\Reporting\\": "src/" }
    },
    "autoload-dev": {
        "psr-4": { "Acme\\Reporting\\Tests\\": "tests/" }
    }
}
```

`require` lists what the package needs to *run*; `require-dev` lists what's
needed only to *develop and test* it (PHPUnit, static analyzers). Anyone
who installs this package as a dependency of their own project pulls in
`require` only — `require-dev` tools never ship to their machine. The same
split applies to `autoload` vs. `autoload-dev`: test helper classes have no
business being autoloadable in production.

```php
<?php
// src/Money.php
declare(strict_types=1);

namespace Acme\Reporting;

final class Money
{
    public function __construct(private int $cents) {}

    public function format(): string
    {
        return '$' . number_format($this->cents / 100, 2);
    }
}
```

```bash
composer dump-autoload
php -r 'require "vendor/autoload.php"; echo (new Acme\Reporting\Money(4250))->format() . "\n";'
```

```text
Generating autoload files
Generated autoload files
$42.50
```

The PSR-4 mapping (`"Acme\\Reporting\\": "src/"`) is what lets
`Acme\Reporting\Money` resolve to `src/Money.php` with no manual `require`
anywhere — Composer generates a class map from the namespace prefix rule,
and `vendor/autoload.php` wires it into PHP's autoloading machinery.

## Version constraints: reading `^`, `~`, and exact pins

| Constraint | Means | Allows |
|---|---|---|
| `^11.0` | Compatible with 11.0, allow non-breaking updates | `>=11.0.0 <12.0.0` |
| `~11.2` | Allow the last listed segment to increase | `>=11.2.0 <11.3.0` |
| `11.4.2` | Exact version | Only that version |
| `>=11.0 <12.0` | Explicit range | Anything in range |
| `dev-main` | Track a branch directly | Whatever `main` currently is (rare outside internal tooling) |

`^` is the default choice for application code — it follows semantic
versioning, meaning a `^11.0` dependency can be safely bumped to `11.9.3`
without Composer asking, because a well-behaved package promises not to
break your code within the same major version. A tighter `~11.2` is for
when you've been burned by a specific minor release and want to hold the
line more conservatively.

## `composer.lock`: why it exists

`composer.json` says "PHPUnit `^11.0`" — a *range*. `composer.lock` records
the *exact* resolved version (`11.5.56`, say) that Composer picked, plus
every transitive dependency's exact version too. `composer install` (used
in CI and on servers) reads the lock file and installs *exactly* those
versions — reproducible, byte-for-byte the same on every machine.
`composer update` is the only command that's allowed to change the lock
file, by re-resolving constraints against the latest available versions.

```bash
# In CI / production: reproducible, uses the lock file, no re-resolution
composer install --no-dev --optimize-autoloader

# On a developer machine, deliberately bumping dependencies:
composer update phpunit/phpunit
```

**Always commit `composer.lock`** for applications (things you deploy).
Library packages meant to be installed *into* other projects typically
`.gitignore` their lock file, since the consuming application's own lock
file is what actually pins versions for that install.

## `composer validate` catches schema mistakes early

```bash
composer validate --strict
```

```text
# Publish errors
- description : The property description is required

# General warnings
- No license specified, it is recommended to do so. For closed-source software you may use "proprietary" as license.
```

Running this in CI (`composer validate --strict --no-check-all`) fails the
build on a malformed `composer.json` before it ever reaches
`composer install` — cheaper to catch a typo in a version constraint here
than to debug a confusing dependency resolution failure later.

## Autoloader performance: `--optimize-autoloader`

PSR-4 autoloading resolves a class name to a file path by checking
namespace prefixes at runtime — fine for development, wasteful for
production where the set of classes never changes between requests.

```bash
composer install --no-dev --optimize-autoloader
```

`--optimize-autoloader` generates a static class map (`ClassName => exact
file path`) once, ahead of time, so production requests skip the prefix
matching entirely — combine with `--classmap-authoritative` to also skip
the filesystem check that falls back to PSR-4 resolution for classes not
in the map, at the cost of requiring `dump-autoload` after adding any new
class.

## PHP traps

**A dependency's dependency (transitive) can silently change your app.**
`composer update` without a package name argument re-resolves *everything*
within its constraints — a transitive dependency three levels down can
jump a minor version and introduce a behavior change you never asked for.
`composer update vendor/package` (naming the specific package) limits the
blast radius; run the full `composer update` deliberately, not as a habit.

**Constraint conflicts fail loudly, but the message is dense.** Requiring
two packages with incompatible transitive constraints on a third produces a
resolution error listing every conflicting rule — read it bottom-up; the
root cause is usually the *last* incompatible pair Composer tried, not the
first line.

**`composer.lock` out of sync with `composer.json` breaks CI, not local
dev.** Editing `composer.json` by hand (adding a `require` line) without
running `composer update <package>` leaves the lock file unaware of the new
dependency; `composer install` in that state either errors
(`composer.lock is not up to date`) with `--no-interaction`, or a plain
`composer install` may still resolve to something unexpected. Never
hand-edit `composer.json` without following up with an update for the
package you touched.

## Composer cheat sheet

| Command | Purpose |
|---|---|
| `composer install` | Install exact versions from `composer.lock` (reproducible) |
| `composer update [pkg]` | Re-resolve constraints, rewrite `composer.lock` |
| `composer require pkg:^1.0` | Add a dependency and update `composer.json` + lock together |
| `composer require --dev pkg` | Add a dev-only dependency |
| `composer dump-autoload` | Regenerate the autoloader after adding classes manually |
| `composer validate --strict` | Catch `composer.json` schema problems in CI |
| `composer install --no-dev --optimize-autoloader` | Production install: no dev deps, static class map |
| `composer show` | List installed packages and versions |

## Exercise

Create a fresh `composer.json` for a package `acme/text-utils` with a
`Acme\TextUtils\` PSR-4 mapping to `src/`, `require-dev` on
`phpunit/phpunit:^11.0`, and a `src/Slugify.php` class with a static
`make(string $text): string` method that lowercases and replaces
non-alphanumeric runs with `-` (trim leading/trailing `-`). Run
`composer dump-autoload`, then a one-line PHP script that requires
`vendor/autoload.php` and asserts
`Acme\TextUtils\Slugify::make("Hello, World!  PHP 8.5") === "hello-world-php-8-5"`.
Finally run `composer validate --strict` and fix whatever warnings appear.
