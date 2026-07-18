# 04 · Functions

Functions group reusable logic under a name. PHP functions can take typed
parameters, default values, and return typed values (all optional, but
recommended).

## Basic function syntax

```php
<?php
function add($a, $b) {
    return $a + $b;
}

echo add(2, 3) . "\n";   // 5
```

## Type declarations and return types

```php
<?php
function add(int $a, int $b): int {
    return $a + $b;
}

echo add(2, 3) . "\n";   // 5
// add("2", "3") would still work due to PHP's type coercion in non-strict mode
```

Add `declare(strict_types=1);` as the very first line of a file to make PHP
reject mismatched types instead of silently coercing them — recommended for
any serious project.

```php
<?php
declare(strict_types=1);

function add(int $a, int $b): int {
    return $a + $b;
}

echo add(2, 3) . "\n";   // 5
// add("2", "3") now throws a TypeError instead of coercing
```

## Default and named arguments

```php
<?php
function greet(string $name, string $greeting = "Hello"): string {
    return "$greeting, $name!";
}

echo greet("Ada") . "\n";               // Hello, Ada!
echo greet("Ada", "Hi") . "\n";         // Hi, Ada!

// Named arguments (PHP 8+) -- skip earlier optional params by name
echo greet(name: "Grace", greeting: "Hey") . "\n";   // Hey, Grace!
```

## Variadic functions (...$args)

```php
<?php
function total(int ...$numbers): int {
    return array_sum($numbers);
}

echo total(1, 2, 3, 4) . "\n";   // 10
```

## Passing by reference

```php
<?php
function doubleIt(int &$n): void {
    $n *= 2;   // modifies the CALLER's variable directly
}

$value = 5;
doubleIt($value);
echo $value . "\n";   // 10
```

By default, PHP passes scalars (int, string, bool, float) by value — the
function gets its own copy. The `&` before the parameter switches to
pass-by-reference, letting the function modify the caller's variable.

## Anonymous functions and arrow functions

```php
<?php
// Anonymous function (closure)
$square = function (int $n): int {
    return $n * $n;
};
echo $square(5) . "\n";   // 25

// Arrow function (PHP 7.4+) -- implicitly captures outer variables
$factor = 3;
$multiply = fn($n) => $n * $factor;
echo $multiply(4) . "\n";   // 12

// Closures over outer variables explicitly, with "use"
$threshold = 10;
$isAboveThreshold = function (int $n) use ($threshold): bool {
    return $n > $threshold;
};
var_dump($isAboveThreshold(15));   // bool(true)
```

`fn() => ...` auto-captures variables from the surrounding scope by value;
regular `function() use (...) {}` closures require you to list captured
variables explicitly.

## Scope

```php
<?php
$counter = 0;   // global scope

function increment() {
    global $counter;   // must explicitly import globals -- unlike JS/Python
    $counter++;
}

increment();
increment();
echo $counter . "\n";   // 2
```

Unlike JavaScript or Python, PHP functions do **not** see outer-scope
variables automatically — you must `global` them or pass them in explicitly.
This is a deliberate design choice that avoids accidental variable leakage.

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Basic function | `function name($a, $b) { ... return $x; }` |
| Typed params/return | `function name(int $a): string { ... }` |
| Default value | `function name($a, $b = 10) { ... }` |
| Named argument | `name(b: 5)` |
| Variadic | `function name(...$args) { ... }` |
| By reference | `function name(&$x) { ... }` |
| Anonymous function | `$f = function($x) { return $x; };` |
| Arrow function | `$f = fn($x) => $x * 2;` |

## Exercise

Write a function `formatCurrency(float $amount, string $symbol = "$"): string`
that returns the amount formatted to 2 decimal places with the symbol
prefixed (e.g. `formatCurrency(9.5)` returns `"$9.50"`). Then write a
variadic function `sumPositive(int ...$numbers): int` that sums only the
positive numbers passed in, ignoring zero and negative values.
