# 09 · Composer & Package Basics

Real PHP projects rarely write everything from scratch — they pull in
well-tested libraries for logging, HTTP requests, date handling, and more.
**Composer** is PHP's dependency manager: it downloads the packages your
project needs, tracks exact versions, and generates an autoloader so you can
`use` classes from those packages without manually `require`-ing every file.

## Why Composer matters

Without a dependency manager you'd have to manually download library code,
track which version you have, resolve conflicts between libraries that need
different versions of a shared dependency, and hand-write `require`
statements for every class file. Composer automates all of that from a single
`composer.json` manifest.

## Installing Composer

Composer is a separate command-line tool, not part of the PHP install itself.

```bash
# macOS / Linux -- one common way, via Homebrew
brew install composer

# Verify it installed
composer --version
# Composer version 2.7.x ...
```

(On Windows, the official installer is `Composer-Setup.exe` from
getcomposer.org; on Linux without Homebrew, the site provides a shell script
installer.)

## `composer init` — starting a new project

```bash
mkdir my-project && cd my-project
composer init
```

`composer init` walks you through an interactive wizard (package name,
description, author, license, dependencies) and writes a `composer.json`
file. You can also just write `composer.json` by hand for a simple project:

```json
{
    "name": "acme/my-project",
    "description": "A small demo project",
    "type": "project",
    "require": {
        "php": ">=8.2"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

## Anatomy of `composer.json`

| Key | Purpose |
|-----|---------|
| `name` | `vendor/package` identifier (lowercase, slash-separated) |
| `description` | Short summary of the project |
| `require` | Production dependencies, with version constraints |
| `require-dev` | Dependencies only needed for testing/development |
| `autoload` | Maps namespaces to directories (PSR-4) so your own classes autoload too |
| `scripts` | Named shell commands you can run via `composer run-script` |

## Requiring a package

```bash
composer require monolog/monolog
```

This does three things: downloads `monolog/monolog` (and its own
dependencies) into `vendor/`, adds an entry to `composer.json`'s `require`
section, and records the exact resolved versions in `composer.lock` so every
machine that runs `composer install` gets identical versions.

```json
{
    "require": {
        "php": ">=8.2",
        "monolog/monolog": "^3.7"
    }
}
```

## Semantic versioning constraints

Composer version constraints follow semantic versioning (`MAJOR.MINOR.PATCH`):
a major bump means breaking changes, minor adds features compatibly, patch
fixes bugs compatibly.

| Constraint | Meaning |
|------------|---------|
| `"3.7.0"` | Exactly this version |
| `"^3.7"` | `>=3.7.0 <4.0.0` — allow minor/patch upgrades, not a new major |
| `"~3.7"` | `>=3.7.0 <3.8.0` — allow patch upgrades only |
| `">=3.0"` | Any version 3.0 or newer, including future majors |
| `"3.7.\*"` | Any patch version within `3.7` |

`^` (caret) is the most common default — it lets `composer update` pull in
bug fixes and new features without risking a breaking major-version change.

## The `vendor/` directory and the autoloader

After `composer require` (or `composer install`), Composer creates a
`vendor/` directory containing all downloaded packages plus a generated
autoloader at `vendor/autoload.php`. Including that one file gives you access
to every installed package's classes — and your own PSR-4-mapped classes too
— without a single manual `require`.

```php
<?php
// index.php
require 'vendor/autoload.php';   // one line, wires up every class in vendor/ (and your own, if PSR-4 mapped)

use Monolog\Logger;
use Monolog\Handler\StreamHandler;

// Create a logger channel named "app"
$log = new Logger('app');
$log->pushHandler(new StreamHandler(__DIR__ . '/app.log', Logger::DEBUG));

$log->info('Application started');
$log->warning('Disk space running low', ['freeMb' => 512]);
$log->error('Failed to connect to payment gateway');

echo "Check app.log for the log output.\n";
```

Running `php index.php` writes structured log lines to `app.log`, something
like:

```text
[2026-07-18T10:15:32+00:00] app.INFO: Application started [] []
[2026-07-18T10:15:32+00:00] app.WARNING: Disk space running low {"freeMb":512} []
[2026-07-18T10:15:32+00:00] app.ERROR: Failed to connect to payment gateway [] []
```

`vendor/autoload.php` should always be `.gitignore`d along with `vendor/`
itself — anyone cloning the project runs `composer install` (which reads
`composer.lock`) to regenerate it identically, so it never needs to be
committed.

## Autoloading your own classes with PSR-4

Composer's autoloader isn't just for third-party packages — map your own
namespace to a directory and Composer will autoload your classes too:

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

```php
<?php
// src/Greeter.php
namespace App;

class Greeter
{
    public function greet(string $name): string
    {
        return "Hello, $name!";
    }
}
```

```php
<?php
// index.php
require 'vendor/autoload.php';

use App\Greeter;

$greeter = new Greeter();
echo $greeter->greet('World');   // Hello, World!
```

After adding or changing an `autoload` mapping, run `composer dump-autoload`
to regenerate the class map.

## Composer commands cheat sheet

| Command | Purpose |
|---------|---------|
| `composer init` | Interactively create a new `composer.json` |
| `composer require <pkg>` | Add and install a production dependency |
| `composer require --dev <pkg>` | Add and install a dev-only dependency |
| `composer install` | Install exact versions from `composer.lock` |
| `composer update` | Resolve and install the newest versions allowed by constraints |
| `composer dump-autoload` | Regenerate the autoloader (e.g. after adding a new class) |
| `composer show` | List installed packages and their versions |

## Exercise

Create a new folder, run `composer init` (or hand-write a minimal
`composer.json` targeting `php >= 8.2`), then run `composer require
ramsey/uuid`. Write a small `index.php` that requires `vendor/autoload.php`,
generates a UUID with `Ramsey\Uuid\Uuid::uuid4()`, and prints it. Confirm
`vendor/` and `composer.lock` were created, and note which section of
`composer.json` now lists `ramsey/uuid` as a dependency.
