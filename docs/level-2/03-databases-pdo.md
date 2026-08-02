# 03 · Working with Databases (PDO)

Almost every real web application needs to store and query data in a
database. **PDO** (PHP Data Objects) is PHP's built-in database abstraction
layer — one consistent API works across MySQL, PostgreSQL, SQLite, and
others, and it's the safe way to build SQL queries that include user input.
The examples below use SQLite (via the `pdo_sqlite` extension, bundled with
PHP) so you can run every snippet with nothing but `php` installed — no
separate database server required.

## Connecting with PDO

```php
<?php

// "sqlite::memory:" creates a throwaway database that lives only for this
// script's run -- perfect for examples and tests. A real app would use
// something like "mysql:host=localhost;dbname=app" or a file path.
$pdo = new PDO("sqlite::memory:");

// Tell PDO to throw exceptions on failure instead of returning false --
// almost always what you want, and NOT the default.
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

$pdo->exec("
    CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT NOT NULL UNIQUE
    )
");

echo "Table created.\n";
```

Without `PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION`, a failed query just
returns `false` and sets an error code you have to remember to check after
*every single call* — easy to forget, and failures fail silently. Exception
mode turns that into a normal `try`/`catch` failure you can't accidentally
ignore.

## Prepared statements: the only safe way to include user input

Never build SQL by concatenating strings with variables — that's how **SQL
injection** happens, where a malicious value changes the meaning of your
query. Prepared statements separate the SQL structure from the data, so user
input can never be interpreted as SQL syntax.

```php
<?php

// DANGEROUS -- never do this with untrusted input:
// $pdo->query("SELECT * FROM users WHERE email = '$email'");
// If $email were:  x' OR '1'='1
// the query becomes: SELECT * FROM users WHERE email = 'x' OR '1'='1'
// which matches every row, not zero.

// SAFE -- the placeholder and the value are sent separately; the driver
// never treats $email as part of the SQL text, no matter what it contains.
$stmt = $pdo->prepare("INSERT INTO users (name, email) VALUES (:name, :email)");
$stmt->execute(["name" => "Ada Lovelace", "email" => "ada@example.com"]);
$stmt->execute(["name" => "Alan Turing", "email" => "alan@example.com"]);

echo "Inserted " . $pdo->lastInsertId() . " total rows so far.\n";
// Inserted 2 total rows so far.
```

`:name` and `:email` are **named placeholders**; PDO also supports
positional `?` placeholders with an ordered array. Either way, the database
driver receives the query text and the values as two separate things — that
separation is what makes injection impossible, not "escaping" the input.

## Querying and fetching results

```php
<?php

$stmt = $pdo->prepare("SELECT id, name, email FROM users WHERE email = :email");
$stmt->execute(["email" => "ada@example.com"]);

// fetch() returns one row (or false if no more rows). PDO::FETCH_ASSOC
// gives you a plain associative array keyed by column name.
$user = $stmt->fetch(PDO::FETCH_ASSOC);
echo "{$user['id']}: {$user['name']}\n";
// 1: Ada Lovelace

// fetchAll() returns every matching row at once as an array of arrays --
// fine for small result sets, wasteful for huge ones.
$all = $pdo->query("SELECT name FROM users ORDER BY name")->fetchAll(PDO::FETCH_ASSOC);
foreach ($all as $row) {
    echo $row['name'] . "\n";
}
// Ada Lovelace
// Alan Turing
```

| Fetch mode | Returns each row as |
|------------|----------------------|
| `PDO::FETCH_ASSOC` | Associative array, keyed by column name |
| `PDO::FETCH_NUM` | Indexed array, keyed by column position |
| `PDO::FETCH_OBJ` | A plain `stdClass` object with columns as properties |
| `PDO::FETCH_CLASS` | An instance of a class you specify, one property set per column |

`$pdo->query()` (no placeholders) is fine for static SQL with no variables
at all; the moment a value comes from outside your source code, use
`prepare()` + `execute()` instead — even if you're "sure" it's safe, since
that assumption is exactly how injection bugs slip in later.

## Reusing a prepared statement in a loop

Preparing once and executing many times is both safer and faster than
preparing a new statement for every row, since the database only has to plan
the query once.

```php
<?php

$stmt = $pdo->prepare("INSERT INTO users (name, email) VALUES (:name, :email)");

$newUsers = [
    ["name" => "Grace Hopper", "email" => "grace@example.com"],
    ["name" => "Katherine Johnson", "email" => "katherine@example.com"],
];

foreach ($newUsers as $user) {
    $stmt->execute($user);   // same prepared statement, different values each time
}

$count = $pdo->query("SELECT COUNT(*) FROM users")->fetchColumn();
echo "Total users: $count\n";
// Total users: 4
```

`fetchColumn()` is a shortcut for grabbing a single scalar value (like a
`COUNT(*)`) without building an array just to read one field.

## Transactions

A transaction groups several statements so they all succeed together or none
of them take effect — critical when one logical operation touches multiple
tables (e.g. debiting one account and crediting another).

```php
<?php

function transferCredits(PDO $pdo, int $fromId, int $toId, int $amount): void
{
    $pdo->beginTransaction();

    try {
        $debit = $pdo->prepare("UPDATE accounts SET balance = balance - :amt WHERE id = :id");
        $debit->execute(["amt" => $amount, "id" => $fromId]);

        $credit = $pdo->prepare("UPDATE accounts SET balance = balance + :amt WHERE id = :id");
        $credit->execute(["amt" => $amount, "id" => $toId]);

        $pdo->commit();   // both updates become permanent together
    } catch (PDOException $e) {
        $pdo->rollBack();   // undo the debit too -- never leave it half-done
        throw $e;
    }
}
```

Without a transaction, a crash or thrown exception between the debit and the
credit would leave money permanently subtracted from one account and never
added to the other.

## Handling errors

```php
<?php

try {
    // UNIQUE constraint on email -- this will fail, since ada@example.com exists
    $stmt = $pdo->prepare("INSERT INTO users (name, email) VALUES (:name, :email)");
    $stmt->execute(["name" => "Duplicate Ada", "email" => "ada@example.com"]);
} catch (PDOException $e) {
    echo "Insert failed: " . $e->getMessage() . "\n";
    // Insert failed: SQLSTATE[23000]: Integrity constraint violation: ...
}
```

With `PDO::ERRMODE_EXCEPTION` set, every PDO failure — a bad query, a
constraint violation, a lost connection — surfaces as a catchable
`PDOException`, so it fits the same error-handling patterns from the
[previous module](02-error-handling-advanced.md) instead of needing manual
`if ($result === false)` checks after every call.

## PDO cheat sheet

| Method | Purpose |
|--------|---------|
| `new PDO($dsn, $user, $pass)` | Open a connection |
| `->prepare($sql)` | Compile a parameterized query, returns a `PDOStatement` |
| `->execute($params)` | Run a prepared statement with bound values |
| `->query($sql)` | Run static SQL with no parameters, returns results directly |
| `->exec($sql)` | Run SQL that doesn't return rows (DDL, bulk `UPDATE`/`DELETE`) |
| `->lastInsertId()` | ID generated by the most recent `INSERT` |
| `->beginTransaction()` / `->commit()` / `->rollBack()` | Group statements atomically |
| `$stmt->fetch()` / `->fetchAll()` / `->fetchColumn()` | Read query results |

## Exercise

Create an in-memory SQLite database with a `products` table (`id`, `name`,
`price`). Write a function `addProduct(PDO $pdo, string $name, float $price):
int` that uses a prepared statement and returns the new row's ID. Write
`findProductsUnder(PDO $pdo, float $maxPrice): array` that safely queries by
a price threshold and returns matching rows as associative arrays. Add a few
products, call `findProductsUnder()`, and print the results — then try
passing a name containing a single quote (like `O'Brien's Widget`) to prove
the prepared statement handles it correctly with no manual escaping.
