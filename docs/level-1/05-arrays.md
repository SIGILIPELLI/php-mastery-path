# 05 · Arrays

PHP has one array type that flexibly acts as both a list (indexed array) and
a dictionary (associative array) — a huge amount of everyday PHP code is just
array manipulation.

## Indexed arrays

```php
<?php
$fruits = ["apple", "banana", "cherry"];

echo $fruits[0] . "\n";   // apple
echo $fruits[2] . "\n";   // cherry
echo count($fruits) . "\n";   // 3

$fruits[] = "date";        // append to the end
$fruits[1] = "blueberry";   // overwrite an existing index

print_r($fruits);
// [0] => apple, [1] => blueberry, [2] => cherry, [3] => date
```

## Associative arrays (key => value)

```php
<?php
$person = [
    "name" => "Ada",
    "age" => 30,
];

echo $person["name"] . "\n";   // Ada

$person["email"] = "ada@example.com";   // add a new key
unset($person["age"]);                   // remove a key

print_r($person);
// [name] => Ada, [email] => ada@example.com
```

## Iterating with foreach

```php
<?php
$prices = ["apple" => 1.50, "banana" => 0.75];

foreach ($prices as $name => $price) {
    echo "$name: \$$price\n";
}
// apple: $1.5
// banana: $0.75
```

## Common array functions

```php
<?php
$numbers = [5, 3, 8, 1, 9];

sort($numbers);                     // sorts in place, ascending
print_r($numbers);                  // [1, 3, 5, 8, 9]

$doubled = array_map(fn($n) => $n * 2, $numbers);
print_r($doubled);                  // [2, 6, 10, 16, 18]

$evens = array_filter($numbers, fn($n) => $n % 2 === 0);
print_r($evens);                    // preserves original keys: [1] => 8 (etc.)

$total = array_reduce($numbers, fn($carry, $n) => $carry + $n, 0);
echo $total . "\n";                 // 26

echo in_array(8, $numbers) ? "found\n" : "not found\n";   // found
echo array_search(8, $numbers) . "\n";   // 3 -- the KEY where 8 lives
```

`array_filter` keeps the original keys, which often leaves gaps in numeric
indexing — call `array_values()` afterward if you need a clean, re-indexed
list.

## Slicing, merging, combining

```php
<?php
$a = [1, 2, 3];
$b = [4, 5, 6];

$merged = array_merge($a, $b);          // [1, 2, 3, 4, 5, 6]
$slice = array_slice($merged, 1, 3);     // [2, 3, 4] -- start at index 1, take 3

$keys = ["a", "b", "c"];
$values = [1, 2, 3];
$combined = array_combine($keys, $values);
print_r($combined);   // [a] => 1, [b] => 2, [c] => 3
```

## Checking structure

```php
<?php
$data = ["name" => "Ada", "age" => 30];

var_dump(isset($data["name"]));    // bool(true)  -- key exists AND isn't null
var_dump(array_key_exists("age", $data));   // bool(true) -- key exists even if value is null
var_dump(empty($data));            // bool(false) -- array has elements
```

`isset()` returns `false` for a key whose value is `null`, even though the
key technically exists — use `array_key_exists()` when that distinction
matters.

## Multi-dimensional arrays

```php
<?php
$students = [
    ["name" => "Ada", "grade" => 92],
    ["name" => "Grace", "grade" => 88],
];

foreach ($students as $student) {
    echo "{$student['name']}: {$student['grade']}\n";
}
// Ada: 92
// Grace: 88
```

## Cheat sheet

| Task | Function |
|------|----------|
| Count elements | `count($arr)` |
| Sort ascending | `sort($arr)` (indexed) / `asort($arr)` (assoc, keeps keys) |
| Transform each element | `array_map($fn, $arr)` |
| Keep matching elements | `array_filter($arr, $fn)` |
| Reduce to a single value | `array_reduce($arr, $fn, $initial)` |
| Check value exists | `in_array($value, $arr)` |
| Check key exists | `array_key_exists($key, $arr)` / `isset($arr[$key])` |
| Merge arrays | `array_merge($a, $b)` |
| Re-index after filter | `array_values($arr)` |

## Exercise

Given an indexed array of associative arrays representing products (each with
`name` and `price` keys), write code that: filters to only products under
$20 using `array_filter`, maps the result to just the names using
`array_map`, and computes the total price of all products using
`array_reduce`.
