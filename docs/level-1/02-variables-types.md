# 02 · Variables & Types

PHP variables always start with a `$` sigil, need no declared type up front,
and can hold any type — the type lives with the *value*, not the variable.
This is called **dynamic typing**.

## Declaring and assigning variables

```php
<?php
$name = "Alice";        // string
$age = 30;               // int
$height = 1.68;          // float (double-precision)
$isStudent = false;      // bool

echo $name . " is " . $age . " years old.\n";
// Alice is 30 years old.

// Variables are case-sensitive: $Name and $name are different variables.
// Variable names must start with a letter or underscore, followed by
// letters, digits, or underscores -- no spaces or hyphens.
```

There's no `int`/`String`/etc. keyword before the variable, and the same
variable can be reassigned to a different type at any point:

```php
<?php
$value = 42;
var_dump($value);   // int(42)

$value = "now a string";
var_dump($value);   // string(13) "now a string"
```

## Scalar types

PHP has four scalar (single-value) types:

```php
<?php
$count = 10;             // int
$price = 19.99;          // float
$label = "T-shirt";      // string
$inStock = true;         // bool

echo gettype($count) . "\n";   // integer
echo gettype($price) . "\n";   // double  (PHP's internal name for float)
echo gettype($label) . "\n";   // string
echo gettype($inStock) . "\n"; // boolean
```

## Inspecting values: `var_dump` and `gettype`

`var_dump()` is the workhorse debugging tool — it prints a value's type
*and* its content, and can take multiple arguments:

```php
<?php
$items = ["apple", "banana"];
var_dump($items);
// array(2) {
//   [0]=>
//   string(5) "apple"
//   [1]=>
//   string(6) "banana"
// }

var_dump(3.14, "hi", null, true);
// float(3.14)
// string(2) "hi"
// NULL
// bool(true)
```

`gettype()` returns just the type name as a string, useful in conditionals or
quick checks — `var_dump()` is better for actually debugging a value.

## Strings: single vs. double quotes

```php
<?php
$name = "World";

// Single quotes: literal text, no interpolation, no escape sequences
// (except \\ and \')
echo 'Hello, $name!\n';     // Hello, $name!\n   (printed literally)

// Double quotes: variables are interpolated, and escape sequences work
echo "Hello, $name!\n";     // Hello, World!

// Concatenation with the "." operator works with either quote style
echo 'Hello, ' . $name . "!\n";   // Hello, World!

// Curly braces disambiguate a variable inside a larger string
$price = 5;
echo "Total: \${$price}.00\n";   // Total: $5.00
```

## Type juggling

PHP automatically converts between types when an operation expects a
different one — this is called **type juggling**:

```php
<?php
$result = "5" + 3;        // int(8)   -- numeric string treated as a number
$result2 = "5" . 3;       // string("53") -- "." forces string concatenation
$result3 = "5 apples" + 3; // int(8) with a warning -- leading numeric part is used
// $result4 = "apples" + 3; // Fatal error (TypeError) in PHP 8+ -- no numeric part at all

echo (1 == "1") ? "equal\n" : "not equal\n";   // equal -- loose comparison juggles types
echo (0 == "abc") ? "equal\n" : "not equal\n"; // not equal in PHP 8+ (changed from PHP 7!)
```

PHP 8 tightened these rules considerably compared to PHP 7 (e.g., `0 == "abc"`
used to be `true`), but juggling still happens — which is exactly why `===`
(covered next) is almost always the safer choice for comparisons.

## Type casting

You can force a conversion explicitly instead of relying on juggling:

```php
<?php
$str = "42";
$num = (int) $str;        // 42 (int)

$float = (float) "3.14";  // 3.14 (float)
$text = (string) 100;     // "100" (string)
$flag = (bool) 0;         // false -- 0, 0.0, "", "0", [], and null are all "falsy"
$flag2 = (bool) "hello";  // true  -- any non-empty string besides "0" is truthy

// Casting to int truncates, it doesn't round
$truncated = (int) 9.9;   // 9

// intval()/floatval()/strval()/boolval() are function equivalents of casts
$n = intval("100 apples"); // 100 -- reads leading numeric portion
```

## `null` and the null coalescing operator

`null` represents "no value." Accessing an undefined variable or array key
returns `null` (with a warning) rather than crashing:

```php
<?php
$user = ["name" => "Bob"];

// isset() checks a variable/key exists AND isn't null
if (isset($user['email'])) {
    echo $user['email'];
} else {
    echo "no email on file\n";
}

// The null coalescing operator "??" is shorthand for the isset() check above:
$email = $user['email'] ?? "no email on file";
echo $email . "\n";   // no email on file

// Null coalescing assignment "??=" -- assign only if currently null/unset
$config = [];
$config['timeout'] ??= 30;
echo $config['timeout'] . "\n";   // 30
```

## Constants

Constants can't be changed once defined and don't use the `$` sigil:

```php
<?php
define('MAX_USERS', 100);      // classic function-based constant
const APP_NAME = 'MyApp';      // "const" keyword -- must be a compile-time value

echo MAX_USERS . "\n";   // 100
echo APP_NAME . "\n";    // MyApp

// MAX_USERS = 200;   // Fatal error -- constants cannot be reassigned
```

`const` is evaluated at compile time and is the more common modern style at
the top level and inside classes; `define()` can compute its value at
runtime (e.g., `define('BUILD_TIME', time())`) and works in any scope.

## Operators

```php
<?php
// Arithmetic
echo 7 + 3, "\n";   // 10
echo 7 - 3, "\n";   // 4
echo 7 * 3, "\n";   // 21
echo 7 / 2, "\n";   // 3.5   -- "/" always does float division if not evenly divisible
echo 7 % 3, "\n";   // 1     -- modulo (remainder)
echo 2 ** 8, "\n";  // 256   -- exponentiation

// String concatenation uses "." (NOT "+")
echo "foo" . "bar", "\n";   // foobar

// Comparison: "==" checks value equality (with juggling), "===" checks
// value AND type equality (no juggling) -- prefer "===" by default
var_dump(1 == "1");    // true  -- loose: string is juggled to int
var_dump(1 === "1");   // false -- strict: int vs string, different types
var_dump(1 === 1);     // true

var_dump(1 != "2");    // true
var_dump(1 !== "1");   // true -- strict "not equal"

echo 5 > 3 ? "yes" : "no", "\n";    // yes
echo 5 <=> 3, "\n";                  // 1  -- spaceship operator: -1, 0, or 1
```

## Cheat sheet

| Concept | Syntax | Example |
|---------|--------|---------|
| Variable | `$name = value;` | `$age = 30;` |
| Integer | `int` | `$n = 42;` |
| Float | `float` / `double` | `$pi = 3.14;` |
| String | `string` | `$s = "hi";` |
| Boolean | `bool` | `$ok = true;` |
| Null | `null` | `$x = null;` |
| Check type | `gettype($v)` | `"integer"` |
| Debug value + type | `var_dump($v)` | `int(42)` |
| Cast | `(int)`, `(float)`, `(string)`, `(bool)` | `(int) "42"` |
| Loose equality | `==` | juggles types |
| Strict equality | `===` | no juggling |
| Null coalescing | `??` | `$v ?? "default"` |
| Null coalescing assign | `??=` | `$v ??= "default"` |
| Constant (compile-time) | `const NAME = val;` | `const PI = 3.14;` |
| Constant (runtime) | `define('NAME', val);` | `define('PI', 3.14);` |
| Concatenation | `.` | `"a" . "b"` → `"ab"` |
| Spaceship | `<=>` | `5 <=> 3` → `1` |

## Exercise

Write `profile.php` that:

1. Declares variables for a person's `$name` (string), `$age` (int),
   `$height` (float in meters), and `$isMember` (bool).
2. Prints a sentence about them using double-quoted string interpolation.
3. Uses `var_dump()` to show the type of each variable.
4. Reads a value that might not exist (e.g., `$profile['nickname']` from an
   empty array `$profile = [];`) using `??` to supply a default, and prints
   the result.
