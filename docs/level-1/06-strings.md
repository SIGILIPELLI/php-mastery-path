# 06 · Strings & String Functions

Strings are one of the most-used types in PHP, especially since PHP spends so
much of its life gluing together HTML, database results, and user input. PHP
gives you several ways to build a string and a huge standard library of
functions for working with them.

## Single quotes vs. double quotes

```php
<?php

$name = "Alice";

// Single quotes: literal text. No interpolation, minimal escaping.
$single = 'Hello, $name! Line 1\nLine 2';
echo $single;
// Hello, $name! Line 1\nLine 2   -- $name and \n are NOT interpreted

// Double quotes: variables are interpolated, escape sequences work.
$double = "Hello, $name! Line 1\nLine 2";
echo $double;
// Hello, Alice! Line 1
// Line 2

// Only \\ and \' need escaping in single-quoted strings.
echo 'It\'s a \\test\\';   // It's a \test\
```

Use single quotes by default for plain text (it's marginally faster and makes
intent clear), and double quotes when you need interpolation or escape
sequences like `\n` or `\t`.

## Interpolation in double-quoted strings

```php
<?php

$user = ['name' => 'Bob', 'age' => 25];
$items = ['apple', 'banana'];

// Simple variable interpolation
echo "User: $user[name]";        // User: Bob (array key, no quotes, no ->)

// Complex interpolation with curly braces -- needed for object properties,
// method calls, or expressions
class Product {
    public function __construct(public string $title, public float $price) {}
}
$product = new Product('Widget', 9.99);

echo "Buying {$product->title} for \${$product->price}";
// Buying Widget for $9.99

echo "First item: {$items[0]}";
// First item: apple
```

Curly-brace interpolation `{$expr}` is the safe, unambiguous form — use it
whenever plain `$var` interpolation looks confusing, especially with array
keys, object properties, or method calls.

## Heredoc and Nowdoc

Heredoc behaves like a double-quoted string (interpolates), nowdoc behaves
like a single-quoted string (literal). Both are useful for multi-line text
without a wall of escaped quotes.

```php
<?php

$name = "Alice";
$role = "Admin";

// Heredoc -- interpolates variables, ends with a bare identifier (no indent
// before PHP 7.3; modern PHP allows the closing marker to be indented)
$message = <<<EOT
    Hello, $name!
    Your role is: $role
    This spans multiple lines.
    EOT;

echo $message;
// Hello, Alice!
// Your role is: Admin
// This spans multiple lines.

// Nowdoc -- literal text, no interpolation (note the single quotes around EOT)
$template = <<<'EOT'
    Hello, $name!
    This $stays literal.
    EOT;

echo $template;
// Hello, $name!
// This $stays literal.
```

Heredoc is common for building multi-line SQL queries or HTML fragments
without escaping every quote.

## Concatenation

```php
<?php

$first = "Jane";
$last = "Doe";

// The . operator concatenates strings
$fullName = $first . " " . $last;
echo $fullName;   // Jane Doe

// .= appends to an existing string
$greeting = "Hello";
$greeting .= ", ";
$greeting .= $fullName;
$greeting .= "!";
echo $greeting;   // Hello, Jane Doe!
```

## Common string functions

```php
<?php

$text = "  Hello, PHP World!  ";

echo strlen($text);              // 22 -- length in bytes
echo strtolower($text);          // "  hello, php world!  "
echo strtoupper($text);          // "  HELLO, PHP WORLD!  "
echo trim($text);                // "Hello, PHP World!" -- strips whitespace from both ends
echo ltrim($text);                // strips only from the left
echo rtrim($text);                // strips only from the right

$clean = trim($text);
echo str_replace("PHP", "Amazing PHP", $clean);
// Hello, Amazing PHP World!

echo substr($clean, 7, 3);       // "PHP" -- start at index 7, take 3 chars
echo substr($clean, -5);         // "World" -- negative index counts from the end

echo strpos($clean, "PHP");      // 7 -- index of first occurrence, or false if not found
var_dump(str_contains($clean, "World")); // bool(true) -- PHP 8+, clearer than strpos !== false
var_dump(str_starts_with($clean, "Hello")); // bool(true) -- PHP 8+
var_dump(str_ends_with($clean, "!"));       // bool(true) -- PHP 8+
```

`strpos` returns `false` (not `-1`) when nothing is found, which is why you
must compare with `!==` rather than a plain truthy check — index `0` is a
valid match but is falsy. The PHP 8 additions `str_contains`, `str_starts_with`,
and `str_ends_with` sidestep that pitfall entirely and should be preferred.

```php
<?php

// Splitting and joining
$csv = "apple,banana,cherry";
$fruits = explode(",", $csv);
print_r($fruits);
// Array ( [0] => apple [1] => banana [2] => cherry )

$joined = implode(" | ", $fruits);
echo $joined;   // apple | banana | cherry

// Formatting strings
$name = "Alice";
$score = 92.5;

$formatted = sprintf("%s scored %.1f%%", $name, $score);
echo $formatted;   // Alice scored 92.5%

printf("%-10s | %5d\n", "Bob", 7);   // "Bob        |     7"
// %-10s = left-align string in 10 chars, %5d = right-align int in 5 chars

// Padding
echo str_pad("5", 3, "0", STR_PAD_LEFT);    // "005" -- common for zero-padded IDs
echo str_pad("hi", 6, "-", STR_PAD_RIGHT);  // "hi----"
```

## Multibyte strings (`mb_*` functions)

PHP's built-in string functions (`strlen`, `substr`, `strtoupper`, ...) treat
strings as raw bytes. That's a problem for text containing multibyte UTF-8
characters (accented letters, emoji, non-Latin scripts) — `strlen` will count
*bytes*, not *characters*.

```php
<?php

$name = "José";

echo strlen($name);     // 5 -- é is 2 bytes in UTF-8, so this is wrong as a char count
echo mb_strlen($name);  // 4 -- correct character count

echo strtoupper($name);    // "JOSé" -- doesn't touch the multibyte character
echo mb_strtoupper($name); // "JOSÉ" -- correctly uppercases it

$greeting = "こんにちは";
echo mb_substr($greeting, 0, 2); // "こん" -- safe multibyte substring
```

Rule of thumb: if your app handles anything beyond plain ASCII (names,
addresses, user-generated content), prefer the `mb_*` equivalents —
`mb_strlen`, `mb_substr`, `mb_strtoupper`, `mb_strtolower`, `mb_str_split` —
over their plain counterparts.

## String functions cheat sheet

| Function | Purpose | Example result |
|----------|---------|-----------------|
| `strlen($s)` | Byte length | `strlen("hi")` → `2` |
| `strtolower($s)` / `strtoupper($s)` | Case conversion | `strtoupper("hi")` → `"HI"` |
| `trim($s)` | Strip whitespace from both ends | `trim(" hi ")` → `"hi"` |
| `str_replace($search, $replace, $s)` | Replace all occurrences | `str_replace("a","o","cat")` → `"cot"` |
| `substr($s, $start, $len)` | Extract a substring | `substr("hello", 1, 3)` → `"ell"` |
| `strpos($s, $needle)` | Index of first match, or `false` | `strpos("hello","l")` → `2` |
| `str_contains($s, $needle)` | Does it contain? (PHP 8+) | `str_contains("hello","ell")` → `true` |
| `explode($sep, $s)` | Split into an array | `explode(",", "a,b")` → `["a","b"]` |
| `implode($sep, $arr)` | Join into a string | `implode("-", ["a","b"])` → `"a-b"` |
| `sprintf($fmt, ...)` | Build a formatted string | `sprintf("%05d", 7)` → `"00007"` |
| `str_pad($s, $len, $pad)` | Pad to a target length | `str_pad("7",3,"0",STR_PAD_LEFT)` → `"007"` |
| `mb_strlen($s)` | Character count (multibyte-safe) | `mb_strlen("José")` → `4` |

## Exercise

Write a function `slugify(string $title): string` that turns a title into a
URL-friendly slug: trim whitespace, lowercase it, replace any run of
non-alphanumeric characters with a single hyphen, and trim leading/trailing
hyphens. For example, `slugify("  Hello, PHP World!  ")` should return
`"hello-php-world"`. Test it with at least three different inputs, including
one with multiple consecutive spaces/punctuation marks.
