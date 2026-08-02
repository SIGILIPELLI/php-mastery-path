# 05 · Testing with PHPUnit

Manually running a script and eyeballing the output doesn't scale — it's
slow, easy to forget, and tells you nothing about whether *yesterday's* code
still works after *today's* change. **PHPUnit** is PHP's standard automated
testing framework: you write small, self-checking functions once, then run
the whole suite in seconds, forever, for free.

## Installing PHPUnit

PHPUnit is installed as a dev dependency via [Composer](../level-1/09-composer-basics.md) —
it's only needed while developing, never in production.

```bash
composer require --dev phpunit/phpunit
```

```json
{
    "require-dev": {
        "phpunit/phpunit": "^11.0"
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    }
}
```

## The code under test

```php
<?php
// src/Calculator.php
declare(strict_types=1);

namespace App;

class Calculator
{
    public function add(float $a, float $b): float
    {
        return $a + $b;
    }

    public function divide(float $a, float $b): float
    {
        if ($b === 0.0) {
            throw new \InvalidArgumentException("Cannot divide by zero");
        }
        return $a / $b;
    }
}
```

## Writing a test case

A test class extends `PHPUnit\Framework\TestCase`; every public method
whose name starts with `test` (or carries a `#[Test]` attribute) is run as
one independent test.

```php
<?php
// tests/CalculatorTest.php
declare(strict_types=1);

namespace Tests;

use App\Calculator;
use PHPUnit\Framework\TestCase;

class CalculatorTest extends TestCase
{
    private Calculator $calculator;

    // Runs fresh before EVERY test method -- guarantees each test starts
    // from a clean, identical object with no leftover state from another test.
    protected function setUp(): void
    {
        $this->calculator = new Calculator();
    }

    public function testAddReturnsTheSum(): void
    {
        $result = $this->calculator->add(2, 3);

        $this->assertSame(5.0, $result);
    }

    public function testDivideReturnsTheQuotient(): void
    {
        $result = $this->calculator->divide(10, 4);

        $this->assertEqualsWithDelta(2.5, $result, 0.0001);
    }

    public function testDivideByZeroThrows(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage("Cannot divide by zero");

        $this->calculator->divide(10, 0);
    }
}
```

Running `vendor/bin/phpunit tests` executes all three and prints a summary:

```text
PHPUnit 11.x by Sebastian Bergmann and contributors.

...                                                                 3 / 3 (100%)

Time: 00:00.012, Memory: 8.00 MB

OK (3 tests, 4 assertions)
```

## Common assertions

| Assertion | Passes when |
|-----------|-------------|
| `assertSame($expected, $actual)` | Values are equal AND same type (`===`) |
| `assertEquals($expected, $actual)` | Values are loosely equal (`==`) |
| `assertEqualsWithDelta($e, $a, $delta)` | Numbers are equal within a tolerance (for floats) |
| `assertTrue($value)` / `assertFalse($value)` | Value is exactly `true` / `false` |
| `assertNull($value)` | Value is `null` |
| `assertCount($n, $array)` | Array/Countable has exactly `$n` elements |
| `assertInstanceOf($class, $obj)` | Object is an instance of `$class` |
| `expectException($class)` | The test body throws `$class` (or a subtype) |

Prefer `assertSame()` over `assertEquals()` by default — `assertEquals()`
uses loose comparison and can pass for surprising reasons (like `"5" ==
5.0`), silently hiding bugs a strict test would catch.

## Data providers: one test, many inputs

Repeating near-identical test methods for different inputs is a maintenance
trap. A data provider runs the same test body once per row of data.

```php
<?php
// tests/CalculatorTest.php (additional test)

use PHPUnit\Framework\Attributes\DataProvider;

class CalculatorTest extends TestCase
{
    // ... testAddReturnsTheSum() etc. from above ...

    #[DataProvider("additionCases")]
    public function testAddWithVariousInputs(float $a, float $b, float $expected): void
    {
        $this->assertSame($expected, (new \App\Calculator())->add($a, $b));
    }

    public static function additionCases(): array
    {
        return [
            "two positives" => [2, 3, 5],
            "negative plus positive" => [-4, 10, 6],
            "two negatives" => [-2, -3, -5],
            "zero plus zero" => [0, 0, 0],
        ];
    }
}
```

This runs `testAddWithVariousInputs` four times, once per array in
`additionCases()`, and reports each case's array key ("two positives", etc.)
individually in the output if one fails — far clearer than one test with
four manual assertions and no indication which pairing broke.

## Testing that depends on collaborators: a simple test double

When a class depends on something slow or external (a database, an HTTP
client), tests should not actually hit it. PHPUnit can create a **mock**
object that stands in for a real dependency and returns canned responses.

```php
<?php

interface PriceLookup
{
    public function getPrice(string $sku): float;
}

class Checkout
{
    public function __construct(private PriceLookup $prices) {}

    public function total(array $skus): float
    {
        $sum = 0.0;
        foreach ($skus as $sku) {
            $sum += $this->prices->getPrice($sku);
        }
        return $sum;
    }
}
```

```php
<?php
// tests/CheckoutTest.php

class CheckoutTest extends TestCase
{
    public function testTotalSumsPricesFromLookup(): void
    {
        // A test double that implements PriceLookup without touching a
        // real database -- fast, deterministic, no network required.
        $prices = $this->createMock(PriceLookup::class);
        $prices->method("getPrice")->willReturnMap([
            ["WIDGET", 9.99],
            ["GADGET", 19.99],
        ]);

        $checkout = new Checkout($prices);

        // Floats: never assertSame() a computed sum against a literal --
        // binary floating-point rounding can make 9.99 + 19.99 differ from
        // the literal 29.98 in its very last bit. A small delta avoids that.
        $this->assertEqualsWithDelta(29.98, $checkout->total(["WIDGET", "GADGET"]), 0.0001);
    }
}
```

Depending on an interface (`PriceLookup`), not a concrete class, is what
makes this mockable — see [OOP Deep Dive](01-oop-deep-dive.md) for why
coding against interfaces pays off beyond just testing.

## PHPUnit cheat sheet

| Command / concept | Purpose |
|--------------------|---------|
| `vendor/bin/phpunit tests` | Run every test in the `tests/` directory |
| `vendor/bin/phpunit --filter testName` | Run only tests matching a name |
| `setUp()` / `tearDown()` | Runs before / after each test method |
| `#[DataProvider("method")]` | Run one test body across many input rows |
| `$this->createMock(Interface::class)` | Create a fake collaborator for isolated testing |
| `phpunit.xml` | Config file: test suite locations, bootstrap file, coverage settings |

## Exercise

Write a `StringHelper` class with a method `slugify(string $text): string`
that lowercases text, replaces runs of non-alphanumeric characters with a
single hyphen, and trims leading/trailing hyphens (e.g. `"Hello, World!"` →
`"hello-world"`). Write a `StringHelperTest` with at least four cases via a
`#[DataProvider]` covering: normal text, text with multiple punctuation
marks in a row, leading/trailing punctuation, and an empty string. Run the
suite with `vendor/bin/phpunit` and confirm all cases pass.
