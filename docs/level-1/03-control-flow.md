# 03 · Control Flow

Control flow statements decide which code runs, and how many times. PHP's
`if`/loops look close to C/Java/JavaScript, plus a modern `match` expression
introduced in PHP 8.

## if / elseif / else

```php
<?php
$score = 82;

if ($score >= 90) {
    echo "Grade: A\n";
} elseif ($score >= 80) {
    echo "Grade: B\n";
} elseif ($score >= 70) {
    echo "Grade: C\n";
} else {
    echo "Grade: F\n";
}
// Grade: B
```

Conditions are evaluated top to bottom; the first truthy branch runs and the
rest are skipped. Remember PHP's "falsy" values: `false`, `0`, `0.0`, `""`,
`"0"`, `[]`, and `null` are all falsy — everything else is truthy.

## switch

```php
<?php
$day = "Wed";

switch ($day) {
    case "Mon":
    case "Tue":
    case "Wed":
    case "Thu":
    case "Fri":
        echo "Weekday\n";
        break;   // without break, execution "falls through" to the next case
    case "Sat":
    case "Sun":
        echo "Weekend\n";
        break;
    default:
        echo "Not a valid day\n";
}
// Weekday
```

`switch` uses loose (`==`) comparison, and forgetting `break` is a classic
bug source — execution keeps running into the next case. This is exactly
what `match` was designed to fix.

## match (PHP 8+) — a safer, expression-based alternative

```php
<?php
$day = "Wed";

// match uses strict (===) comparison, requires no "break", and is an
// EXPRESSION -- it evaluates to a value you can assign or return directly.
$type = match ($day) {
    "Mon", "Tue", "Wed", "Thu", "Fri" => "Weekday",
    "Sat", "Sun" => "Weekend",
    default => "Not a valid day",
};

echo $type . "\n";   // Weekday

// "match (true)" + boolean arms -- a common trick for range checks,
// since match compares each arm to the subject with "===" until one is true
$score = 82;

$grade = match (true) {
    $score >= 90 => "A",
    $score >= 80 => "B",
    $score >= 70 => "C",
    default => "F",
};

echo $grade . "\n";   // B

// match with no matching arm and no default throws UnhandledMatchError
```

| | `switch` | `match` |
|---|----------|---------|
| Comparison | Loose (`==`) | Strict (`===`) |
| Fall-through | Yes, unless `break` | No — each arm is isolated |
| Returns a value | No (it's a statement) | Yes (it's an expression) |
| No match / no default | Just does nothing | Throws `UnhandledMatchError` |

## while and do-while

```php
<?php
// while: checks the condition BEFORE each iteration
$count = 0;
while ($count < 3) {
    echo "count is $count\n";
    $count++;
}
// count is 0 / count is 1 / count is 2

// do-while: runs the body ONCE, then checks -- guarantees at least one run
$attempts = 0;
do {
    echo "attempt $attempts\n";
    $attempts++;
} while ($attempts < 3);
// attempt 0 / attempt 1 / attempt 2
```

## for loops

```php
<?php
// for (initializer; condition; increment)
for ($i = 1; $i <= 5; $i++) {
    echo $i . " ";
}
echo "\n";
// 1 2 3 4 5

// Multiple expressions can be comma-separated in each slot
for ($i = 0, $j = 10; $i < 3; $i++, $j--) {
    echo "i=$i j=$j\n";
}
// i=0 j=10 / i=1 j=9 / i=2 j=8
```

## foreach — the idiomatic way to loop over arrays

```php
<?php
$fruits = ["apple", "banana", "cherry"];

// Value only
foreach ($fruits as $fruit) {
    echo $fruit . "\n";
}
// apple / banana / cherry

// Key and value
$prices = ["apple" => 1.50, "banana" => 0.75, "cherry" => 3.00];
foreach ($prices as $name => $price) {
    echo "$name costs \$$price\n";
}
// apple costs $1.5 / banana costs $0.75 / cherry costs $3

// Modify by reference with "&" -- without it, foreach works on a COPY
$numbers = [1, 2, 3];
foreach ($numbers as &$n) {
    $n *= 10;
}
unset($n);   // best practice: break the reference after the loop
print_r($numbers);   // [10, 20, 30]
```

`foreach` doesn't use an index variable you manage yourself — it's the
preferred loop for arrays in idiomatic PHP, reached for far more often than
`for`.

## break and continue

```php
<?php
foreach ([1, 2, 3, 4, 5, 6] as $n) {
    if ($n === 4) {
        break;      // exits the loop entirely
    }
    echo $n . " ";
}
echo "\n";
// 1 2 3

foreach ([1, 2, 3, 4, 5, 6] as $n) {
    if ($n % 2 === 0) {
        continue;   // skips to the next iteration
    }
    echo $n . " ";
}
echo "\n";
// 1 3 5

// A numeric argument breaks/continues out of that many nested loop levels
for ($i = 0; $i < 3; $i++) {
    for ($j = 0; $j < 3; $j++) {
        if ($j === 1) {
            continue 2;   // skips to the next iteration of the OUTER loop
        }
        echo "i=$i j=$j  ";
    }
}
echo "\n";
// i=0 j=0  i=1 j=0  i=2 j=0
```

## Ternary and null coalescing in control flow

```php
<?php
$age = 20;

// Ternary: condition ? if_true : if_false
$status = $age >= 18 ? "adult" : "minor";
echo $status . "\n";   // adult

// Short ternary "?:" -- returns the left side if it's truthy, else the right
$nickname = "";
$displayName = $nickname ?: "Anonymous";   // "" is falsy, so we get "Anonymous"
echo $displayName . "\n";

// Null coalescing "??" only checks for null/unset (not falsiness) -- covered
// in Module 2, but worth contrasting with "?:" here:
$value = null;
echo ($value ?? "default") . "\n";   // default
echo ($value ?: "default") . "\n";   // default -- same result here, but
                                       // "0" or "" would differ between the two
```

## Cheat sheet

| Construct | Use when |
|-----------|----------|
| `if / elseif / else` | General branching on any condition |
| `switch` | Comparing one value against several possibilities (legacy style) |
| `match` | Comparing one value against several possibilities (PHP 8+, strict, expression) |
| `while` | Loop while a condition holds, check-first |
| `do-while` | Loop at least once, check-last |
| `for` | Loop a known number of times with a counter |
| `foreach` | Loop over every element of an array (idiomatic default) |
| `break` | Exit the current loop (or `break N` for N levels) |
| `continue` | Skip to the next iteration (or `continue N` for N levels) |
| `?:` | Short ternary, falls back on falsy values |
| `??` | Null coalescing, falls back only on null/unset |

## Exercise

Write `fizzbuzz.php` that loops from 1 to 30 with a `for` loop and, for each
number `$i`:

- Uses `match` (not `if`/`switch`) to decide what to print: `"FizzBuzz"` if
  divisible by both 3 and 5, `"Fizz"` if divisible by 3 only, `"Buzz"` if
  divisible by 5 only, otherwise the number itself.
- Skips printing anything for multiples of 13 using `continue`.
- Stops the loop entirely once it reaches 25 using `break`.
