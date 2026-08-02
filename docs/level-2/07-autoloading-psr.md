# 07 · Autoloading & PSR Standards

[Level 1's Composer module](../level-1/09-composer-basics.md) showed the
practical side: add a `psr-4` block to `composer.json`, run `composer
dump-autoload`, and classes load without manual `require`. This module goes
one level deeper — how autoloading actually works under the hood, what "PSR"
even means, and the autoloading mistakes that produce the most confusing
"Class not found" errors.

## Autoloading before Composer: `spl_autoload_register`

Composer's autoloader isn't magic — it's built on a plain PHP function.
`spl_autoload_register()` registers a callback that PHP calls automatically
whenever code references a class that hasn't been loaded yet, giving your
callback a chance to `require` the right file just in time.

```php
<?php
// A tiny hand-rolled autoloader, no Composer involved -- this is
// conceptually what Composer's generated autoloader does for you.
spl_autoload_register(function (string $className): void {
    // Convert "App\Models\User" into "src/Models/User.php"
    $relativePath = str_replace("\\", "/", $className) . ".php";
    $path = __DIR__ . "/src/" . str_replace("App/", "", $relativePath);

    if (file_exists($path)) {
        require $path;
    }
});

// No `require` anywhere above this line -- PHP calls the callback the
// moment "new App\Models\User" is evaluated, because User isn't defined yet.
$user = new App\Models\User();
```

You'll rarely write one of these by hand in a real project — Composer's
generated `vendor/autoload.php` calls `spl_autoload_register()` for you,
with a much more complete implementation — but knowing this exists explains
*why* `require 'vendor/autoload.php';` at the top of a script is enough to
make every class in the project available.

## PSR-4: the namespace-to-directory mapping rule

**PSR-4** is a published standard (from PHP-FIG, the PHP Framework
Interop Group) that defines exactly how a fully-qualified class name maps to
a file path, so every tool and library can agree on the same convention.

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "App\\Tests\\": "tests/"
        }
    }
}
```

| Fully-qualified class name | Maps to file |
|------------------------------|---------------|
| `App\Models\User` | `src/Models/User.php` |
| `App\Http\Controllers\HomeController` | `src/Http/Controllers/HomeController.php` |
| `App\Tests\UserTest` | `tests/UserTest.php` |

The rule: strip the mapped namespace prefix (`App\`), replace every
remaining `\` with `/`, append `.php`, and resolve relative to the mapped
directory (`src/`). This is why PSR-4 requires **one class per file** and the
class name to exactly match the filename — the mapping is purely mechanical,
with no scanning or configuration per class.

```php
<?php
// src/Models/User.php
declare(strict_types=1);

namespace App\Models;   // must match the directory structure under src/

class User
{
    public function __construct(public string $name) {}
}
```

```php
<?php
// index.php
require "vendor/autoload.php";

use App\Models\User;

$user = new User("Ada");
echo $user->name;   // Ada
```

## Multiple autoloading strategies in one project

PSR-4 covers the common case, but `composer.json` supports a few other
autoload strategies for things that don't fit a clean namespace mapping:

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        },
        "classmap": ["legacy/"],
        "files": ["src/helpers.php"]
    }
}
```

| Strategy | Use for |
|----------|---------|
| `psr-4` | Modern, namespaced classes — one class per file, path mirrors namespace |
| `classmap` | Legacy code with inconsistent naming; Composer scans the directory and indexes every class it finds, regardless of file/folder naming |
| `files` | Plain function files with no classes at all (e.g. a `helpers.php` full of global functions) — always loaded on every request |

`classmap` needs `composer dump-autoload` re-run after adding a new class
file (Composer has to rescan), while `psr-4` classes are found automatically
the first time they're referenced, since the path is computed, not looked up
in a pre-built index.

## Common autoloading errors

```php
<?php
// Error: Class "App\Models\user" not found
// Cause: filesystem is case-sensitive on Linux (the deploy server) even
// though it wasn't on the developer's Mac -- "user.php" != "User.php".
// Fix: match the class name's capitalization exactly in the filename.

// Error: Class "App\Services\Mailer" not found
// Cause: the file's `namespace` declaration doesn't match its location --
// e.g. the file lives at src/Mailer.php but declares `namespace App\Services;`
// PSR-4 requires the namespace to mirror the DIRECTORY, not just be "close".

// Error: Class "App\NewFeature\Widget" not found (right after adding the file)
// Cause: forgetting `composer dump-autoload` after adding a NEW namespace
// prefix (not needed for a new class under an EXISTING psr-4 prefix -- only
// for classmap entries or a brand-new mapping in composer.json itself).
```

## PSR standards beyond autoloading

PSR-4 is just one numbered standard in a series PHP-FIG maintains so
different frameworks and libraries can interoperate without agreeing on
everything by hand:

| Standard | Covers |
|----------|--------|
| PSR-1 | Basic coding standard — file encoding, side-effect-free files, class/method naming conventions |
| PSR-12 | Extended coding style — brace placement, indentation, import ordering (supersedes the older PSR-2) |
| PSR-3 | A common `LoggerInterface` so any PSR-3 logger can be swapped for another |
| PSR-4 | Autoloading — namespace-to-file-path mapping (this module) |
| PSR-7 | HTTP message interfaces (requests/responses) shared across frameworks |

You don't need to memorize the numbers — the point is that when a library
says "PSR-12 compliant" or "accepts any PSR-3 logger," it's promising to
follow one of these shared conventions instead of inventing its own, which
is what lets unrelated packages from different vendors work together.

## Optimizing the autoloader for production

```bash
composer dump-autoload --optimize
```

In development, Composer's autoloader computes each class's file path on
the fly from the PSR-4 rules. `--optimize` (or `-o`) generates a static,
pre-computed class-to-file map instead — faster to look up, at the cost of
needing to be regenerated after adding classes (normally handled
automatically as part of your deployment process, not something you run by
hand in dev).

## Autoloading cheat sheet

| Concept | Summary |
|---------|---------|
| `spl_autoload_register()` | Low-level function Composer's autoloader is built on |
| PSR-4 rule | `Namespace\Prefix\ClassName` → `mapped/dir/ClassName.php` |
| One class per file | Required for PSR-4's mechanical path computation to work |
| `classmap` | Scans and indexes classes that don't follow PSR-4 |
| `files` | Always-loaded plain function files |
| `composer dump-autoload` | Regenerate the autoloader after structural changes |
| `composer dump-autoload -o` | Production-optimized, pre-computed class map |

## Exercise

Create a project with `src/Http/Request.php` declaring `namespace
App\Http; class Request { ... }` and a `composer.json` mapping `"App\\":
"src/"`. Confirm it autoloads correctly with `require
"vendor/autoload.php"; new App\Http\Request();`. Then rename the class
inside the file to `Reguest` (typo) without renaming the file, run your
script again, and read the exact "Class not found" error PHP produces —
notice it names the class you tried to use, not the mismatched file, which
is why this particular typo is easy to misdiagnose.
