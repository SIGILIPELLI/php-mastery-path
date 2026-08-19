# 05 · Performance Optimization at Scale

[Level 3's caching module](../level-3/05-performance-caching.md) covered
OPcache and application-level caching. At scale, two more patterns matter
more than either: the number of database round-trips a single request
makes (the classic **N+1 query problem**), and how much memory a request
holds onto while processing large datasets (where **generators** replace
building giant arrays in memory).

## The N+1 query problem

Loading a list, then looping over it to fetch each item's related data
separately, turns "one page load" into "1 + N database round-trips" — each
one paying full network latency to the database, even on a fast local
connection.

```php
<?php
declare(strict_types=1);

$pdo = new PDO('sqlite::memory:');
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$pdo->exec('CREATE TABLE authors (id INTEGER PRIMARY KEY, name TEXT)');
$pdo->exec('CREATE TABLE books (id INTEGER PRIMARY KEY, author_id INTEGER, title TEXT)');
$pdo->exec("INSERT INTO authors VALUES (1,'Ada'),(2,'Grace'),(3,'Alan')");
$pdo->exec("INSERT INTO books VALUES (1,1,'Book A'),(2,1,'Book B'),(3,2,'Book C'),(4,3,'Book D')");

$queryCount = 0;
function countedQuery(PDO $pdo, string $sql, array $params = []): array
{
    global $queryCount;
    $queryCount++;
    $stmt = $pdo->prepare($sql);
    $stmt->execute($params);
    return $stmt->fetchAll(PDO::FETCH_ASSOC);
}

// N+1: one query for the books, then ONE MORE query per book for its author.
$queryCount = 0;
$books = countedQuery($pdo, 'SELECT * FROM books');
foreach ($books as $book) {
    $author = countedQuery($pdo, 'SELECT * FROM authors WHERE id = ?', [$book['author_id']]);
}
echo "N+1 approach: $queryCount queries for " . count($books) . " books\n";

// Fixed: one query total, using a JOIN.
$queryCount = 0;
$joined = countedQuery(
    $pdo,
    'SELECT books.title, authors.name FROM books JOIN authors ON authors.id = books.author_id'
);
echo "JOIN approach: $queryCount query for " . count($joined) . " rows\n";
foreach ($joined as $row) {
    echo "  {$row['title']} by {$row['name']}\n";
}
```

```text
N+1 approach: 5 queries for 4 books
JOIN approach: 1 query for 4 rows
  Book A by Ada
  Book B by Ada
  Book C by Grace
  Book D by Alan
```

Four books produced five queries — one for the list, four more for each
book's author — while a `JOIN` gets the same data in one round-trip. This
is easy to miss with an ORM: `foreach ($books as $book) { echo $book->author->name; }`
*looks* like plain property access, but if `->author` lazily fires a query
per access (the default in many ORMs unless you explicitly ask for eager
loading — Eloquent's `with('author')`, Doctrine's `JOIN` fetch), the
N+1 pattern is happening invisibly underneath.

## Generators: processing large datasets without holding them all in memory

An array built from a million rows exists entirely in memory at once. A
**generator** produces values one at a time, on demand, using `yield`
instead of `return` — the function's state pauses between values instead
of computing them all up front.

```php
<?php
declare(strict_types=1);

function eagerRange(int $n): array
{
    $out = [];
    for ($i = 0; $i < $n; $i++) {
        $out[] = $i;
    }
    return $out;
}

function lazyRange(int $n): Generator
{
    for ($i = 0; $i < $n; $i++) {
        yield $i;
    }
}

$before = memory_get_usage();
$arr = eagerRange(1_000_000);
$afterEager = memory_get_usage();
unset($arr);

$before2 = memory_get_usage();
$sum = 0;
foreach (lazyRange(1_000_000) as $n) {
    $sum += $n;
}
$afterLazy = memory_get_usage();

printf("Eager array: %.2f MB\n", ($afterEager - $before) / 1024 / 1024);
printf("Lazy generator peak delta: %.2f MB\n", ($afterLazy - $before2) / 1024 / 1024);
echo "Sum via generator: $sum\n";
```

```text
Eager array: 16.02 MB
Lazy generator peak delta: 0.00 MB
Sum via generator: 499999500000
```

`eagerRange()` allocates 16MB to hold a million integers before the loop
even starts summing them. `lazyRange()`'s memory delta rounds to zero
because at any given moment, only the *current* value exists — the
generator's internal state (which `$i` it's on) is tiny, and each yielded
value is consumed and discarded by the `foreach` before the next one is
produced. This is exactly how PDO's own `fetchAll()` (loads everything)
differs from iterating a `PDOStatement` directly with `foreach` (fetches
row by row) — for a genuinely large result set, iterating the statement
directly, or wrapping row fetching in your own generator, avoids ever
holding the whole result set in PHP memory at once.

## Combining both: a generator over a paginated query

```php
<?php
declare(strict_types=1);

function fetchAllBooksLazily(PDO $pdo, int $pageSize = 2): Generator
{
    $offset = 0;
    while (true) {
        $stmt = $pdo->prepare('SELECT * FROM books LIMIT :limit OFFSET :offset');
        $stmt->bindValue('limit', $pageSize, PDO::PARAM_INT);
        $stmt->bindValue('offset', $offset, PDO::PARAM_INT);
        $stmt->execute();
        $page = $stmt->fetchAll(PDO::FETCH_ASSOC);

        if ($page === []) {
            return; // no more rows -- ends the generator
        }
        foreach ($page as $row) {
            yield $row;
        }
        $offset += $pageSize;
    }
}

$count = 0;
foreach (fetchAllBooksLazily($pdo) as $book) {
    $count++;
    echo "  processing: {$book['title']}\n";
}
echo "Processed $count books, 2 rows fetched per database round-trip\n";
```

```text
  processing: Book A
  processing: Book B
  processing: Book C
  processing: Book D
Processed 4 books, 2 rows fetched per database round-trip
```

This is the shape a real "export 10 million rows to CSV" job takes: page
through the table in bounded chunks (never `SELECT *` with no `LIMIT` on a
huge table), yield each row, and let the caller's `foreach` process one row
at a time — constant memory regardless of whether the table has 4 rows or
4 million.

## PHP traps

**A generator can only be iterated once.** `foreach ($gen as $x) {}` a
second time over the *same* generator instance throws
`Exception: Cannot rewind a generator that was already run` — unlike an
array, which can be looped repeatedly. If a value needs to be consumed more
than once, either call the generator-returning function again (fresh
instance) or materialize it into an array first (accepting the memory
cost you were trying to avoid).

**`fetchAll()` inside a loop that's supposed to be memory-efficient
defeats the whole point.** The paginated generator above calls
`fetchAll()` *per page* (a small, bounded amount), not once for the entire
table — the memory savings come from bounding the page size, not from
avoiding `fetchAll()` entirely.

**An eager `JOIN` fix for N+1 can itself become slow if it's not
selective.** Joining a table with millions of rows against another large
table without an index on the join column produces a table scan, not a
fast lookup — fixing N+1 without checking `EXPLAIN QUERY PLAN` (SQLite) or
`EXPLAIN` (MySQL/Postgres) on the resulting join can trade "many small slow
queries" for "one enormous slow query." Both need to be measured, not
assumed.

## Performance-at-scale cheat sheet

| Problem | Symptom | Fix |
|---|---|---|
| N+1 queries | 1 query becomes 1+N as a list grows | `JOIN`, or eager-load related data up front |
| Large array in memory | `memory_get_usage()` spikes with dataset size | Generator (`yield`) instead of building an array |
| Unbounded `SELECT *` | Query time and memory both grow with table size | `LIMIT`/`OFFSET` pagination, processed via a generator |
| Slow `JOIN` | Fixing N+1 introduces a new slow query | Check `EXPLAIN`/`EXPLAIN QUERY PLAN`; add indexes on join columns |
| Generator reused | `Cannot rewind a generator` exception | Call the generator function again, or materialize once into an array |

## Exercise

Add a `countBooksByAuthorLazily(PDO $pdo): Generator` that yields
`['author' => $name, 'count' => $n]` pairs using a single `GROUP BY` query
(no N+1), and a *deliberately naive* `countBooksByAuthorNPlus1(PDO $pdo,
array $authors): array` that loops over each author and runs a separate
`SELECT COUNT(*) FROM books WHERE author_id = ?` query. Instrument both with
the `countedQuery()` pattern from this module, run both against the sample
`authors`/`books` data, print the query count for each, and confirm they
report the same counts per author despite the very different number of
queries used to get there.
