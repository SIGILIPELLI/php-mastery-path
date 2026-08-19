# 03 · Testing at Scale & CI

A handful of tests run in a fraction of a second, as every example so far
in this path has shown. Hundreds or thousands of tests across a real
codebase need organization — configuration instead of ad-hoc `phpunit`
flags, grouping so a slow subset can be skipped locally, and a CI pipeline
that runs the whole suite automatically on every push so a broken test
never reaches production unnoticed.

## `phpunit.xml`: configuration instead of flags

Passing `--testdox tests` on the command line every time doesn't scale —
`phpunit.xml` (or `phpunit.xml.dist`, committed to the repo) makes the
configuration part of the project itself, so `vendor/bin/phpunit` alone
does the right thing for anyone who checks it out.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit bootstrap="vendor/autoload.php" colors="true">
  <testsuites>
    <testsuite name="unit">
      <directory>tests</directory>
    </testsuite>
  </testsuites>
  <groups>
    <exclude>
      <group>slow</group>
    </exclude>
  </groups>
</phpunit>
```

```bash
vendor/bin/phpunit --testdox
```

```text
PHPUnit 11.5.56 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.5.9
Configuration: /path/to/project/phpunit.xml

......                                                              6 / 6 (100%)

Time: 00:00.011, Memory: 8.00 MB

Task Api (TaskApi\Tests\TaskApi)
 ✔ Creating a task returns 201 with the stored task
 ✔ Creating a task with blank title returns 422
 ✔ Index lists all created tasks
 ✔ Show returns 404 for missing task
 ✔ Complete marks task done
 ✔ Destroy removes the task

OK (6 tests, 13 assertions)
```

`bootstrap="vendor/autoload.php"` means every test file can reference any
autoloaded class with no manual `require` — this is what made
`TaskApi\TaskController` just work in the [REST API project](../level-3/10-project-rest-api.md)'s
tests without a single `require` statement in the test file itself.

## Grouping tests: separating fast feedback from full coverage

Not every test needs to run on every save. A `#[Group('slow')]` attribute
(PHPUnit 11's modern replacement for the older `@group` doc-comment
annotation, which is deprecated) marks a test as excludable, and
`phpunit.xml`'s `<groups><exclude>` (above) skips it by default.

```php
<?php
// tests/TaskApiTest.php (excerpt)
declare(strict_types=1);

namespace TaskApi\Tests;

use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\TestCase;

final class TaskApiTest extends TestCase
{
    // ... other tests from the REST API project ...

    #[Group('slow')]
    public function testSlowIntegrationExample(): void
    {
        usleep(1000); // stand-in for something genuinely slow: a real DB, an HTTP call
        $this->assertTrue(true);
    }
}
```

```bash
vendor/bin/phpunit --testdox           # excludes 'slow' via phpunit.xml -- 6 tests run
vendor/bin/phpunit --group slow        # runs ONLY the slow group -- 1 test runs
```

```text
......                                                              6 / 6 (100%)

Time: 00:00.011, Memory: 8.00 MB

OK (6 tests, 13 assertions)
```

The seventh test (`testSlowIntegrationExample`) never shows up in that
run's count — the `<exclude><group>slow</group></exclude>` block filtered
it out before execution, not just from the output. Fast, everyday tests
(unit tests hitting an in-memory SQLite database, like this project's) run
by default; slower ones (real network calls, a full browser-driven
end-to-end test) opt in explicitly with `--group slow` when you actually
want them.

## Continuous integration: running the suite on every push

A CI pipeline runs the test suite automatically, on infrastructure separate
from any one developer's machine, so "works on my machine" can't hide a
broken build. GitHub Actions is the most common choice for a GitHub-hosted
PHP project — here's a workflow shaped for the REST API project from Level 3:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: "8.3"
          extensions: pdo_sqlite

      - name: Install dependencies
        run: composer install --prefer-dist --no-progress

      - name: Run test suite
        run: vendor/bin/phpunit --testdox

      - name: Run full suite including slow tests on main
        if: github.ref == 'refs/heads/main'
        run: vendor/bin/phpunit --group slow
```

Two design choices worth noting: `composer install` (not `update`) uses the
committed `composer.lock` for a reproducible install identical to what a
developer would get locally — see [Composer at Scale](../level-3/09-composer-at-scale.md).
And the slow-group tests only run on `main`, not on every pull request —
fast feedback on PRs, full coverage before anything actually merges to the
default branch.

## Code coverage: knowing what your tests don't touch

`vendor/bin/phpunit --coverage-text` reports which lines your test suite
actually executes, using a coverage driver (Xdebug or PCOV) installed
alongside PHP — neither is installed in the environment these docs are
built in, which is why this module doesn't show a captured coverage run.
The concept matters regardless: 100% coverage doesn't mean "bug-free" (a
line can execute without its result ever being asserted on), but a coverage
report is very good at surfacing code nobody tests at all — an untested
`catch` block, an `if` branch that's never hit. CI pipelines commonly gate
merges on a coverage *floor* (e.g. "don't let coverage drop below 80%")
rather than chasing 100%, since the last few percent are usually
diminishing returns (defensive code, framework glue) rather than real risk.

## PHP traps

**`@group` doc-comment annotations are deprecated in PHPUnit 11+** in favor
of PHP 8 attributes (`#[Group('slow')]`) — the old syntax still works today
but emits a deprecation notice, and mixing the two styles across a codebase
makes `phpunit.xml`'s `<groups>` filtering inconsistent to reason about.
Prefer attributes in any project targeting PHP 8+.

**A CI job that never installs `pdo_sqlite` fails mysteriously**, not with
"extension missing" but with a `PDOException` deep inside test setup — CI
runner images vary in which PHP extensions ship by default, so declaring
`extensions: pdo_sqlite` explicitly (as in the workflow above) avoids a
failure that looks like a broken test rather than a missing dependency.

**Tests that pass locally but flake in CI are usually a hidden shared
state problem**, not bad luck — the [REST API project](../level-3/10-project-rest-api.md)'s
`setUp()` gives every test a *fresh* `:memory:` SQLite database precisely
to avoid this; a test suite sharing one file-based database (or, worse, a
staging server) across parallel CI runs will intermittently fail from
cross-test pollution that's very hard to reproduce locally.

## Testing-at-scale cheat sheet

| Tool | Purpose |
|---|---|
| `phpunit.xml` | Committed test configuration — bootstrap, suites, group filters |
| `#[Group('slow')]` | Marks a test excludable/selectable by name |
| `<groups><exclude>` | Skips a named group by default |
| `--group slow` | Runs *only* the named group |
| `composer install` in CI | Reproducible install from the committed lock file |
| `--coverage-text` (needs Xdebug/PCOV) | Reports which lines the suite actually exercises |
| Coverage floor (e.g. 80%) | A practical CI gate — catches obviously-untested code without chasing 100% |

## Exercise

Add a `#[Group('slow')]` test to the [REST API project](../level-3/10-project-rest-api.md)'s
`TaskApiTest` that creates 50 tasks in a loop and asserts `index()` returns
exactly 50 — representative of a "large dataset" integration test that's
fine to skip on every save but worth running before a merge. Add a
`phpunit.xml` to that project excluding the `slow` group by default, run
`vendor/bin/phpunit --testdox` and confirm the count stays at 6, then run
`vendor/bin/phpunit --group slow` and confirm the new test runs and passes
on its own.
