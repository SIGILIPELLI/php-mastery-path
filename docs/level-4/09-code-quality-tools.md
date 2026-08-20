# 09 · Code Quality Tools

Every module in this path has been about making code *work*. This one is
about the tools that check whether code that already works is also
**correct in ways tests don't catch**, **written consistently**, and
**actually exercised by the test suite** — the three jobs of a static
analyzer, a formatter, and a coverage report, run here for real against a
small, deliberately imperfect codebase.

## The codebase under test

```php
<?php
// src/Cart.php
declare(strict_types=1);

namespace App;

final class Cart
{
    /** @var array<int, array{name: string, price: int, qty: int}> */
    private array $items = [];

    public function add(int $id, string $name, int $priceCents, int $qty = 1): void
    {
        $this->items[$id] = ['name' => $name, 'price' => $priceCents, 'qty' => $qty];
    }

    public function total(): int
    {
        $sum = 0;
        foreach ($this->items as $item) {
            $sum += $item['price'] * $item['qty'];
        }
        return $sum;
    }

    public function isEmpty(): bool
    {
        return count($this->items) === 0;
    }
}
```

```php
<?php
// src/Messy.php -- written carelessly on purpose, to give the tools
// something real to complain about.
declare(strict_types=1);
namespace App;
class Messy {
function   add($a,$b)  {
  return $a+$b;
}
}
```

`Cart` is typed, single-purpose, and formatted the way the rest of this
path has written code all along. `Messy` has no type hints, no visibility
keyword on its method, and inconsistent whitespace — everything the tools
below are built to catch.

## Static analysis: PHPStan

A test suite only tells you about the paths it exercises. **Static
analysis** reads the code without running it and checks that types line up
everywhere, including branches no test ever hits. `phpstan/phpstan`,
installed via Composer, is the standard choice:

```bash
composer require --dev phpstan/phpstan
```

```yaml
# phpstan.neon
parameters:
    level: 6
    paths:
        - src
```

Levels run from 0 (barely checks anything) to 9 (strictest — every
property, parameter, and return type must be fully declared). Level 6 is a
reasonable floor for new code: it demands typed parameters and return
types, without yet requiring the exhaustive generic annotations level 9
wants. Running it against this module's two files:

```bash
vendor/bin/phpstan analyse
```

```text
 ------ ------------------------------------------------------------------
  Line   Messy.php
 ------ ------------------------------------------------------------------
  5      Method App\Messy::add() has no return type specified.
         missingType.return
  5      Method App\Messy::add() has parameter $a with no type specified.
         missingType.parameter
  5      Method App\Messy::add() has parameter $b with no type specified.
         missingType.parameter
 ------ ------------------------------------------------------------------


 [ERROR] Found 3 errors
```

`Cart.php` produces nothing — it's clean at level 6. `Messy.php` fails on
exactly the things its name promises: no declared types anywhere. This is
the kind of bug that a test suite calling `add(2, 3)` would never surface
(it "works" for ints) but that breaks the moment someone calls
`add('2', [3])` and gets a `TypeError` — or worse, silent coercion — in
production instead of a warning at commit time.

## Formatting: PHP-CS-Fixer

Style disagreements (tabs vs. spaces, brace placement, blank-line rules)
waste review time on things a machine should just fix. `friendsofphp/php-cs-fixer`
rewrites code to match a configured rule set:

```php
<?php
// .php-cs-fixer.php
$finder = PhpCsFixer\Finder::create()->in(__DIR__ . '/src');

return (new PhpCsFixer\Config())
    ->setRules(['@PSR12' => true, 'array_syntax' => ['syntax' => 'short']])
    ->setFinder($finder);
```

`--dry-run --diff` shows what *would* change without touching the files —
the mode to run in CI, where you want the build to fail on drift rather
than silently rewrite someone's branch:

```bash
vendor/bin/php-cs-fixer fix --dry-run --diff
```

```text
   1) src/Messy.php
      ---------- begin diff ----------
--- src/Messy.php
+++ src/Messy.php
@@ -1,8 +1,13 @@
 <?php
+
 declare(strict_types=1);
+
 namespace App;
-class Messy {
-function   add($a,$b)  {
-  return $a+$b;
-}
+
+class Messy
+{
+    public function add($a, $b)
+    {
+        return $a + $b;
+    }
 }

      ----------- end diff -----------

Found 2 of 2 files that can be fixed in 0.011 seconds, 18.00 MB memory used
```

Even `Cart.php` — already reasonably tidy — got flagged, for a missing
blank line after `<?php`. That's the point of running a fixer instead of
eyeballing style by hand: a human reviewer's attention drifts after the
tenth PR; the fixer applies PSR-12 the same way on file one and file one
thousand. Dropping `--dry-run --diff` actually rewrites the files in
place.

## Test coverage: PHPUnit

A green test suite answers "do the tests I wrote pass?" — not "how much of
the code do those tests actually reach?" Coverage reporting answers the
second question, line by line:

```php
<?php
// tests/CartTest.php
declare(strict_types=1);

namespace App\Tests;

use App\Cart;
use PHPUnit\Framework\TestCase;

final class CartTest extends TestCase
{
    public function testEmptyCartHasZeroTotal(): void
    {
        $cart = new Cart();
        $this->assertTrue($cart->isEmpty());
        $this->assertSame(0, $cart->total());
    }

    public function testTotalSumsPriceTimesQuantity(): void
    {
        $cart = new Cart();
        $cart->add(1, 'Widget', 500, 3);
        $cart->add(2, 'Gadget', 1200, 1);
        $this->assertSame(2700, $cart->total());
    }
}
```

```bash
vendor/bin/phpunit --testdox
```

```text
Cart (App\Tests\Cart)
 [x] Empty cart has zero total
 [x] Total sums price times quantity

OK (2 tests, 3 assertions)
```

Both tests pass, and between them they exercise every branch of `Cart`:
the empty case, `add()`, and `total()` summing across more than one item.
`Messy` has no test file at all — a coverage report (which needs a driver
like Xdebug or PCOV enabled, unlike the pass/fail run above) would show it
at 0%, flagging exactly the file static analysis already flagged. That
overlap isn't a coincidence: code nobody bothered to type-hint is usually
code nobody bothered to test, either.

## PHP traps

**Running these tools manually means they eventually don't get run.** A
`composer.json` with a `scripts` block —
`"check": ["phpstan analyse", "php-cs-fixer fix --dry-run", "phpunit"]` —
turns three separate commands into `composer check`, and wiring that same
command into CI (as [module 03](03-testing-at-scale-ci.md) covered) is what
actually makes it non-optional rather than a step someone forgets under
deadline pressure.

**A high PHPStan level adopted all at once on an existing codebase produces
thousands of errors and gets disabled in frustration.** The realistic path
is a `baseline` file (`phpstan analyse --generate-baseline`) that freezes
today's existing errors as "known, not blocking," while any *new* code is
held to the full level — the codebase improves incrementally instead of
demanding a rewrite before the tool is useful at all.

**100% coverage is a number, not a guarantee.** A test that calls
`$cart->total()` and asserts nothing about the result *executes* every
line without checking any of them are correct — coverage tools measure
which lines *ran*, not which lines were *verified*. `assertSame(2700, ...)`
above is what makes the second test meaningful; the coverage percentage
alone would have been identical with a bare `$cart->total();` and no
assertion at all.

## Code quality cheat sheet

| Tool | Question it answers | Command used here |
|---|---|---|
| PHPStan | Do the types actually line up? | `phpstan analyse` |
| PHP-CS-Fixer | Does the code match one consistent style? | `php-cs-fixer fix --dry-run --diff` |
| PHPUnit | Do the behaviors I claim work actually work? | `phpunit --testdox` |
| Coverage report | Which lines did the test suite never reach? | `phpunit --coverage-text` (needs Xdebug/PCOV) |
| Baseline | How to adopt strict analysis on old code | `phpstan analyse --generate-baseline` |
| CI wiring | What makes any of this non-optional | `composer check` run in the pipeline |

## Exercise

Add types to `Messy::add()` — `int $a, int $b): int` — rerun
`phpstan analyse` and confirm the 3 errors are gone, then rerun
`php-cs-fixer fix --dry-run --diff` and fix the remaining formatting by
hand (brace placement, spacing) so the diff for `Messy.php` also comes back
empty. Finally, write a `MessyTest` with one test asserting
`(new Messy())->add(2, 3) === 5`, and run `phpunit --testdox` to confirm
all three files in this module — `Cart`, `Messy`, and their tests — are
clean.
