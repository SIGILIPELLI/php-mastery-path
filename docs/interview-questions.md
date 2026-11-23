---
title: "PHP Interview Questions and Answers (178 Questions)"
description: "178 PHP interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the PHP Mastery Path."
---

# PHP Interview Questions and Answers

**178 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free PHP Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What is PHP?

**Answer:** A server-side scripting language widely used for the web

Runs on the server.

**Q2.** What do PHP files start with?

**Answer:** &lt;?php

Opening tag.

**Q3.** How run a script from the CLI?

**Answer:** php script.php

CLI SAPI.

**Q4.** What does the built-in server do?

**Answer:** php -S localhost:8000 serves a directory for development

Not for production.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```php
$a = "5"; $b = 5;
var_dump($a == $b, $a === $b);
echo intdiv(7, 2), " ", 7 % 3, " ", 2 ** 3, " ", 10 / 4;
```

**Answer:** bool(true)
bool(false)
3 1 8 2.5

== coerces types; === doesn't. Verified by compiling and running it.

**Q6.** What does this print?

```php
$x = null;
echo $x ?? "default", " ", $x?->name ?? "none", " ", gettype(1.0), " ", gettype("1"), " ", (int) "12abc";
```

**Answer:** default none double string 12

Null coalescing, nullsafe, casts. Verified by compiling and running it.

**Q7.** What does declare(strict_types=1) do?

**Answer:** Enforces scalar type declarations without coercion in that file

Safer.

**Q8.** Is PHP dynamically typed?

**Answer:** Yes, with optional type declarations

Types are gradual.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```php
$n = 3;
$r = match(true) { $n < 2 => "small", $n < 5 => "medium", default => "big" };
for ($i = 0, $s = 0; $i < 6; $i++) { if ($i % 2) continue; $s += $i; }
echo "$r $s";
```

**Answer:** medium 6

match is strict and returns a value. Verified by compiling and running it.

**Q10.** What does this print?

```php
$i = 0;
while (true) { if (++$i > 3) break; echo $i; }
foreach ([1, 2, 3] as $k => $v) echo "|$k:$v";
```

**Answer:** 123|0:1|1:2|2:3

break and foreach with keys. Verified by compiling and running it.

**Q11.** What's the difference between match and switch?

**Answer:** match uses strict comparison, has no fallthrough and returns a value

PHP 8.

**Q12.** What does the spaceship operator &lt;=&gt; return?

**Answer:** -1, 0 or 1 comparing two values

Sorting callbacks.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```php
function greet(string $name, string $greeting = "Hello"): string { return "$greeting, $name!"; }
function sum(int ...$n): int { return array_sum($n); }
$sq = fn($x) => $x * $x;
echo greet("Ada"), " ", greet(greeting: "Hi", name: "Bo"), " ", sum(1, 2, 3), " ", $sq(4);
```

**Answer:** Hello, Ada! Hi, Bo! 6 16

Defaults, named args, variadics, arrow functions. Verified by compiling and running it.

**Q14.** What does this print?

```php
function inc(int &$n) { $n++; }
$a = 1; inc($a);
$counter = function() { static $c = 0; return ++$c; };
echo $a, " ", $counter(), $counter();
```

**Answer:** 2 12

References and static variables. Verified by compiling and running it.

**Q15.** What does a nullable type ?int mean?

**Answer:** The value may be int or null

PHP 7.1+.

**Q16.** What's the return type void?

**Answer:** The function returns nothing

No value.

### [05 · Arrays](level-1/05-arrays.md)

**Q17.** What does this print?

```php
$a = [3, 1, 2];
sort($a);
$m = ["b" => 2, "a" => 1];
ksort($m);
$m["c"] = 3;
echo implode(",", $a), " ", json_encode($m), " ", count($a), " ", in_array(2, $a) ? "yes" : "no";
```

**Answer:** 1,2,3 {"a":1,"b":2,"c":3} 3 yes

Indexed and associative arrays. Verified by compiling and running it.

**Q18.** What does this print?

```php
$xs = [1, 2, 3, 4, 5];
echo implode(",", array_filter($xs, fn($x) => $x % 2)), " ", implode(",", array_map(fn($x) => $x * 2, $xs)), " ", array_reduce($xs, fn($c, $x) => $c + $x, 0), " ", implode(",", array_slice($xs, 1, 2));
```

**Answer:** 1,3,5 2,4,6,8,10 15 2,3

Functional helpers. Verified by compiling and running it.

**Q19.** What does this print?

```php
[$a, [$b, $c]] = [1, [2, 3]];
["x" => $x] = ["x" => 9];
$m = [...[1, 2], ...[3]];
echo "$a$b$c$x ", count($m);
```

**Answer:** 1239 3

Destructuring and spread. Verified by compiling and running it.

**Q20.** What does array_key_exists vs isset do for a null value?

**Answer:** array_key_exists is true; isset is false

Null handling.

### [06 · Strings &amp; String Functions](level-1/06-strings.md)

**Q21.** What does this print?

```php
$s = "Hello, World";
echo strlen($s), " ", strtoupper($s), " ", substr($s, 7), " ", str_contains($s, "World") ? "yes" : "no", " ", ucfirst("php"), " ", str_pad("7", 3, "0", STR_PAD_LEFT);
```

**Answer:** 12 HELLO, WORLD World yes Php 007

String helpers. Verified by compiling and running it.

**Q22.** What does this print?

```php
$name = "Ada"; $a = ['k' => 5];
echo "Hi $name {$a['k']} ", 'no $name', " ", sprintf("%05.1f|%-4s|%d%%", 3.14159, "ab", 50);
```

**Answer:** Hi Ada 5 no $name 003.1|ab  |50%

Single quotes don't interpolate. Verified by compiling and running it.

**Q23.** What does this print?

```php
echo strlen("héllo"), " ", mb_strlen("héllo"), " ", mb_strtoupper("héllo");
```

**Answer:** 6 5 HÉLLO

strlen counts bytes; use mb_* for UTF-8. Verified by compiling and running it.

**Q24.** Why prefer htmlspecialchars when echoing user text?

**Answer:** To prevent XSS by escaping HTML

Output escaping.

### [07 · Classes &amp; Objects Basics](level-1/07-classes-objects.md)

**Q25.** What does this print?

```php
class Counter { private int $n = 0; public function __construct(private string $label = "c") {} public function inc(): static { $this->n++; return $this; } public function __toString(): string { return "{$this->label}={$this->n}"; } }
$c = (new Counter("hits"))->inc()->inc();
echo $c;
```

**Answer:** hits=2

Constructor promotion and fluent methods. Verified by compiling and running it.

**Q26.** What does this print?

```php
class A { public static int $count = 0; const X = 5; public static function make(): static { static::$count++; return new static; } }
class B extends A {}
$b = B::make();
echo get_class($b), " ", A::$count, " ", B::X, " ", $b instanceof A ? "yes" : "no";
```

**Answer:** B 1 5 yes

Late static binding. Verified by compiling and running it.

**Q27.** What does readonly do?

**Answer:** Allows a property to be assigned only once (immutability)

PHP 8.1+.

**Q28.** What's the difference between private and protected?

**Answer:** protected is visible to subclasses too

Visibility.

### [08 · Error Handling Basics](level-1/08-error-handling.md)

**Q29.** What does this print?

```php
function f() { try { throw new InvalidArgumentException("bad", 42); } catch (RuntimeException $e) { return "runtime"; } catch (InvalidArgumentException | TypeError $e) { return get_class($e) . ":" . $e->getCode(); } finally { echo "[fin]"; } }
echo f();
```

**Answer:** [fin]InvalidArgumentException:42

Multi-catch and finally. Verified by compiling and running it.

**Q30.** What does this print?

```php
function div(int $a, int $b): float { if ($b === 0) throw new DivisionByZeroError("zero"); return $a / $b; }
try { div(1, 0); } catch (Error $e) { echo get_class($e), " ", $e->getMessage(); }
try { intdiv(1, 0); } catch (DivisionByZeroError $e) { echo " | ", $e->getMessage(); }
```

**Answer:** DivisionByZeroError zero | Division by zero

Throwable hierarchy. Verified by compiling and running it.

**Q31.** What's the difference between Exception and Error?

**Answer:** Errors are engine-level problems (TypeError); both implement Throwable

PHP 7+.

**Q32.** What does set_error_handler do?

**Answer:** Converts or handles PHP warnings/notices in your own code

ErrorException.

### [09 · Composer &amp; Package Basics](level-1/09-composer-basics.md)

**Q33.** What is Composer?

**Answer:** PHP's dependency manager

composer.json.

**Q34.** What does composer.lock do?

**Answer:** Pins exact installed versions for reproducible installs

Commit it for apps.

**Q35.** What does the vendor/autoload.php file do?

**Answer:** Autoloads classes from installed packages and your PSR-4 mappings

require once.

**Q36.** What does `^2.1` mean?

**Answer:** Compatible versions &gt;=2.1 &lt;3.0

Caret constraint.

### [10 · Project — CLI To-Do App](level-1/10-project-todo-app.md)

**Q37.** What does the todo app combine?

**Answer:** Classes, arrays, forms/CLI input, file persistence and Composer autoloading

Level 1 project.

**Q38.** How persist todos?

**Answer:** Serialize to JSON and write with file_put_contents

Simple storage.

**Q39.** How validate input?

**Answer:** Filter and validate before use (filter_var, type checks)

Never trust input.

**Q40.** How structure code?

**Answer:** Separate model, storage and presentation classes

Cohesion.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

**Q41.** What does this print?

```php
interface Shape { public function area(): float; }
abstract class Base implements Shape { public function describe(): string { return static::class . " " . number_format($this->area(), 2); } }
final class Circle extends Base { public function __construct(private float $r) {} public function area(): float { return M_PI * $this->r ** 2; } }
final class Rect extends Base { public function __construct(private float $w, private float $h) {} public function area(): float { return $this->w * $this->h; } }
foreach ([new Circle(1), new Rect(2, 3)] as $s) echo $s->describe(), "; ";
```

**Answer:** Circle 3.14; Rect 6.00;

Polymorphism via abstract base and interface. Verified by compiling and running it.

**Q42.** What does this print?

```php
trait Greets { public function hi(): string { return "hi from " . static::class; } }
class A { use Greets; }
enum Suit: string { case Hearts = "H"; case Spades = "S"; public function label(): string { return ucfirst(strtolower($this->name)); } }
echo (new A)->hi(), " ", Suit::from("S")->label(), " ", Suit::tryFrom("X")?->name ?? "none", " ", count(Suit::cases());
```

**Answer:** hi from A Spades none 2

Traits and enums. Verified by compiling and running it.

**Q43.** When use an interface vs a trait?

**Answer:** Interface defines a contract; trait shares implementation

Composition.

**Q44.** What does `final` do?

**Answer:** Prevents extension of a class or overriding of a method

Design intent.

### [02 · Error Handling Advanced](level-2/02-error-handling-advanced.md)

**Q45.** What does this print?

```php
class NotFound extends RuntimeException { public function __construct(public readonly string $id, ?Throwable $prev = null) { parent::__construct("Item $id not found", 404, $prev); } }
try { try { throw new LogicException("db down"); } catch (LogicException $e) { throw new NotFound("A1", $e); } } catch (NotFound $e) { echo $e->getMessage(), " ", $e->getCode(), " ", $e->getPrevious()->getMessage(), " ", $e->id; }
```

**Answer:** Item A1 not found 404 db down A1

Custom exceptions with chaining. Verified by compiling and running it.

**Q46.** What should an app do with uncaught exceptions in production?

**Answer:** Log them and show a generic error page without details

display_errors off.

**Q47.** What's exception chaining for?

**Answer:** Preserving the original cause when wrapping

getPrevious().

**Q48.** What does `finally` guarantee?

**Answer:** It runs whether or not an exception was thrown

Cleanup.

### [03 · Working with Databases (PDO)](level-2/03-databases-pdo.md)

**Q49.** What does this print?

```php
$pdo = new PDO("sqlite::memory:");
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$pdo->exec("CREATE TABLE u(id INTEGER PRIMARY KEY, name TEXT)");
$st = $pdo->prepare("INSERT INTO u(name) VALUES(?)");
foreach (["ada", "bo", "x' OR '1'='1"] as $n) $st->execute([$n]);
$q = $pdo->prepare("SELECT count(*) FROM u WHERE name = :n");
$q->execute([":n" => "x' OR '1'='1"]);
echo $pdo->query("SELECT count(*) FROM u")->fetchColumn(), " ", $q->fetchColumn();
```

**Answer:** 3 1

Prepared statements treat input as data. Verified by compiling and running it.

**Q50.** What does this print?

```php
$pdo = new PDO("sqlite::memory:");
$pdo->exec("CREATE TABLE a(id INTEGER PRIMARY KEY, bal INTEGER)");
$pdo->exec("INSERT INTO a VALUES (1, 100), (2, 0)");
try { $pdo->beginTransaction(); $pdo->exec("UPDATE a SET bal = bal - 50 WHERE id = 1"); throw new Exception("boom"); } catch (Exception $e) { $pdo->rollBack(); }
echo implode(",", $pdo->query("SELECT bal FROM a ORDER BY id")->fetchAll(PDO::FETCH_COLUMN));
```

**Answer:** 100,0

Transactions roll back. Verified by compiling and running it.

**Q51.** Why use PDO?

**Answer:** A consistent database API with prepared statements across drivers

Security and portability.

**Q52.** What does PDO::ATTR_ERRMODE exception do?

**Answer:** Makes failures throw PDOException

Handle errors.

### [04 · Sessions &amp; Forms](level-2/04-sessions-forms.md)

**Q53.** How does PHP track a session?

**Answer:** A session ID (cookie) mapping to server-side data in $_SESSION

session_start().

**Q54.** What should you do after login?

**Answer:** session_regenerate_id(true) to prevent fixation

Security.

**Q55.** What protects forms from CSRF?

**Answer:** A per-session token verified on submission (plus SameSite cookies)

Tokens.

**Q56.** Which cookie flags help?

**Answer:** HttpOnly, Secure and SameSite

XSS/CSRF.

### [05 · Testing with PHPUnit](level-2/05-phpunit-testing.md)

**Q57.** What does PHPUnit provide?

**Answer:** A testing framework with assertions, data providers and mocks

composer require --dev.

**Q58.** What does a data provider do?

**Answer:** Runs one test with multiple input sets

@dataProvider.

**Q59.** What's a good test name?

**Answer:** Describes behavior and expectation

Readable.

**Q60.** What does setUp() do?

**Answer:** Runs before every test to create fresh state

Isolation.

### [06 · Working with JSON/APIs](level-2/06-json-apis.md)

**Q61.** What does this print?

```php
$data = ["name" => "Ada", "tags" => ["a", "b"], "n" => 1.5, "ok" => true, "none" => null];
$json = json_encode($data);
echo $json, " ", json_decode($json, true)["tags"][1], " ", json_decode("{bad", true) === null ? json_last_error_msg() : "ok";
```

**Answer:** {"name":"Ada","tags":["a","b"],"n":1.5,"ok":true,"none":null} b Syntax error

JSON round trip and error handling. Verified by compiling and running it.

**Q62.** What does JSON_THROW_ON_ERROR do?

```php
try { json_decode("{bad", false, 512, JSON_THROW_ON_ERROR); } catch (JsonException $e) { echo get_class($e), ": ", $e->getMessage(); }
```

**Answer:** JsonException: Syntax error

Prefer exceptions over checking json_last_error. Verified by compiling and running it.

**Q63.** What status should an API return for a created resource?

**Answer:** 201 Created

With Location header.

**Q64.** What header declares JSON responses?

**Answer:** Content-Type: application/json

header().

### [07 · Autoloading &amp; PSR Standards](level-2/07-autoloading-psr.md)

**Q65.** What does PSR-4 define?

**Answer:** Autoloading classes by namespace-to-directory mapping

composer autoload.

**Q66.** What does `namespace App\Models;` do?

**Answer:** Declares the class's namespace to avoid name collisions

Fully qualified names.

**Q67.** What does the `use` statement do?

**Answer:** Imports a class name into the current file scope

Aliases too.

**Q68.** Why autoload instead of require?

**Answer:** Classes load on demand without manual includes

Maintainable.

### [08 · File Uploads &amp; Validation](level-2/08-file-uploads-validation.md)

**Q69.** How validate an uploaded file?

**Answer:** Check upload errors, size, real MIME type and extension allowlist

finfo.

**Q70.** Where store uploads?

**Answer:** Outside the web root with generated names

Prevent execution.

**Q71.** What does move_uploaded_file do?

**Answer:** Safely moves a file uploaded via HTTP POST

Checks origin.

**Q72.** What's the danger of trusting the client filename?

**Answer:** Path traversal and executable extensions

Generate names.

### [09 · Working with Dates](level-2/09-working-with-dates.md)

**Q73.** What does this print?

```php
$d = new DateTimeImmutable("2024-01-31 10:00", new DateTimeZone("UTC"));
$n = $d->modify("+1 month");
echo $d->format("Y-m-d"), " ", $n->format("Y-m-d"), " ", $d->setTimezone(new DateTimeZone("Asia/Kolkata"))->format("H:i"), " ", $d->diff(new DateTimeImmutable("2024-03-01 10:00", new DateTimeZone("UTC")))->days;
```

**Answer:** 2024-01-31 2024-03-02 15:30 30

Month overflow and timezone conversion. Verified by compiling and running it.

**Q74.** Why prefer DateTimeImmutable?

**Answer:** Operations return new objects, avoiding accidental mutation

Safer.

**Q75.** How store timestamps?

**Answer:** In UTC, convert for display

Avoid DST bugs.

**Q76.** What does date_default_timezone_set do?

**Answer:** Sets the default timezone for date functions

Configure explicitly.

### [10 · Project — CRUD Web App with SQLite](level-2/10-project-crud-app.md)

**Q77.** What does the CRUD app project combine?

**Answer:** PDO, forms, sessions, validation and tests

Level 2 project.

**Q78.** How prevent duplicate submissions?

**Answer:** Post/Redirect/Get and server-side checks

Idempotence.

**Q79.** How validate and show errors?

**Answer:** Validate server-side and re-render the form with messages

UX.

**Q80.** How protect mutations?

**Answer:** Auth checks, CSRF tokens and authorization per resource

Security.

## Level 3 · Advanced

### [01 · Building APIs](level-3/01-building-apis.md)

**Q81.** What should a good REST API return on invalid input?

**Answer:** 400/422 with a clear error body

Client errors.

**Q82.** How version breaking changes?

**Answer:** New version path or header, supporting old clients during migration

Compatibility.

**Q83.** What's idempotent among HTTP methods?

**Answer:** PUT and DELETE (and GET)

Safe retries.

**Q84.** How structure responses?

**Answer:** Consistent JSON envelopes and error format

Predictable.

### [02 · Design Patterns in PHP](level-3/02-design-patterns.md)

**Q85.** What does this print?

```php
final class Config { private static ?Config $i = null; private function __construct(public array $v = ["env" => "prod"]) {} public static function get(): self { return self::$i ??= new self(); } }
interface Notifier { public function send(string $m): string; }
class Mail implements Notifier { public function send(string $m): string { return "mail:$m"; } }
class Sms implements Notifier { public function send(string $m): string { return "sms:$m"; } }
class Service { public function __construct(private Notifier $n) {} public function run(): string { return $this->n->send("hi"); } }
echo var_export(Config::get() === Config::get(), true), " ", (new Service(new Sms))->run(), " ", (new Service(new Mail))->run();
```

**Answer:** true sms:hi mail:hi

Singleton, strategy and constructor injection. Verified by compiling and running it.

**Q86.** What does the Factory pattern do?

**Answer:** Centralizes object creation behind an interface

Decoupling.

**Q87.** What's the Observer pattern?

**Answer:** Subjects notify subscribers of state changes

Events.

**Q88.** Why avoid singletons?

**Answer:** Hidden global state hampers testing; use DI

Inject.

### [03 · Security](level-3/03-security.md)

**Q89.** What does this print?

```php
$pw = "s3cret";
$h = password_hash($pw, PASSWORD_DEFAULT);
echo password_verify($pw, $h) ? "ok" : "no", " ", password_verify("wrong", $h) ? "ok" : "no", " ", strlen($h) >= 60 ? "long" : "short", " ", htmlspecialchars("<script>alert(1)</script>"), " ", hash_equals("abc", "abc") ? "same" : "diff";
```

**Answer:** ok no long &amp;lt;script&amp;gt;alert(1)&amp;lt;/script&amp;gt; same

password_hash, escaping and constant-time compare. Verified by compiling and running it.

**Q90.** How prevent SQL injection?

**Answer:** Prepared statements with bound parameters

PDO.

**Q91.** How prevent XSS?

**Answer:** Escape output for its context (htmlspecialchars) and set a CSP

Contextual.

**Q92.** Where store secrets?

**Answer:** Environment variables or a secret manager, not in the repo

Config.

### [04 · Testing Advanced (Mocking)](level-3/04-testing-advanced-mocking.md)

**Q93.** What does a mock do in PHPUnit?

**Answer:** Replaces a dependency and verifies interactions

createMock.

**Q94.** What's a stub?

**Answer:** A test double returning canned values

willReturn.

**Q95.** What's over-mocking?

**Answer:** Mocking so much that tests check implementation details

Brittle.

**Q96.** What's a better target than mocking everything?

**Answer:** Test behavior with fakes (in-memory repositories)

Resilient.

### [05 · Performance &amp; Caching](level-3/05-performance-caching.md)

**Q97.** What does OPcache do?

**Answer:** Caches compiled bytecode to skip recompiling scripts

Big win.

**Q98.** What's APCu/Redis for?

**Answer:** Caching application data between requests

Shared cache.

**Q99.** What's the first step in tuning?

**Answer:** Profile (Xdebug/Blackfire) to find hotspots

Evidence.

**Q100.** What commonly slows apps?

**Answer:** N+1 queries and missing indexes

Database.

### [06 · Working with Queues](level-3/06-working-with-queues.md)

**Q101.** Why use a queue for emails?

**Answer:** Keep requests fast and retry failures outside the request

Async work.

**Q102.** What must job handlers be?

**Answer:** Idempotent, since delivery can repeat

Retries.

**Q103.** Which backends are common?

**Answer:** Redis, RabbitMQ, SQS or database queues

Brokers.

**Q104.** What's a dead-letter queue?

**Answer:** Where jobs go after repeated failures

Inspect.

### [07 · Middleware Patterns](level-3/07-middleware-patterns.md)

**Q105.** What is middleware?

**Answer:** Code wrapping request handling (auth, logging, CORS)

PSR-15.

**Q106.** How does the pipeline run?

**Answer:** Each middleware calls the next handler and can act before and after

Onion.

**Q107.** What can middleware short-circuit with?

**Answer:** A response (e.g. 401) without calling the next handler

Early exit.

**Q108.** What does PSR-7 define?

**Answer:** HTTP message interfaces (Request/Response)

Interoperability.

### [08 · Dependency Injection Basics](level-3/08-dependency-injection.md)

**Q109.** What does this print?

```php
class Container { private array $defs = []; private array $shared = []; public function set(string $id, callable $f): void { $this->defs[$id] = $f; } public function get(string $id) { return $this->shared[$id] ??= ($this->defs[$id])($this); } }
class Db {} class Repo { public function __construct(public Db $db) {} }
$c = new Container();
$c->set(Db::class, fn() => new Db);
$c->set(Repo::class, fn($c) => new Repo($c->get(Db::class)));
var_dump($c->get(Repo::class)->db === $c->get(Db::class));
```

**Answer:** bool(true)

A tiny DI container with shared instances. Verified by compiling and running it.

**Q110.** What does dependency injection give you?

**Answer:** Loose coupling and easy substitution in tests

Constructor injection.

**Q111.** What does PSR-11 define?

**Answer:** A container interface (get/has)

Interop.

**Q112.** What's autowiring?

**Answer:** The container resolving constructor dependencies by type

Convenient.

### [09 · Working with Composer Packages at Scale](level-3/09-composer-at-scale.md)

**Q113.** What does `composer install` do in CI?

**Answer:** Installs exactly what composer.lock pins

Reproducible.

**Q114.** What does composer audit do?

**Answer:** Checks dependencies for known vulnerabilities

Supply chain.

**Q115.** What's a private package approach?

**Answer:** Private Packagist/Satis or VCS repositories

Internal libs.

**Q116.** What does `--no-dev` do?

**Answer:** Skips development dependencies for production installs

Smaller.

### [10 · Project — REST API Service](level-3/10-project-rest-api.md)

**Q117.** What does the REST API project combine?

**Answer:** Routing, validation, PDO, auth, errors and tests

Level 3 project.

**Q118.** How test endpoints?

**Answer:** HTTP-level tests with an in-memory DB

Fast.

**Q119.** How handle errors consistently?

**Answer:** A central handler mapping exceptions to JSON responses

Uniform.

**Q120.** How document the API?

**Answer:** OpenAPI spec kept in sync

Contract.

## Level 4 · Master

### [01 · Advanced Framework Patterns](level-4/01-advanced-framework-patterns.md)

**Q121.** What do frameworks like Laravel/Symfony provide?

**Answer:** Routing, DI, ORM, validation and tooling out of the box

Productivity.

**Q122.** What's a service provider?

**Answer:** A place to register services in the container

Bootstrapping.

**Q123.** What's event-driven design for?

**Answer:** Decoupling reactions to domain events

Listeners.

**Q124.** What's a repository pattern?

**Answer:** An abstraction over data access for the domain

Testability.

### [02 · Security at Scale](level-4/02-security-at-scale.md)

**Q125.** What does CSP do?

**Answer:** Restricts allowed script/style sources to mitigate XSS

Header.

**Q126.** What's mass assignment risk?

**Answer:** Binding request data directly to models and overwriting protected fields

Whitelist fields.

**Q127.** What does rate limiting protect?

**Answer:** Login and API endpoints from abuse

Brute force.

**Q128.** What should logs avoid?

**Answer:** Passwords, tokens and personal data

Redact.

### [03 · Testing at Scale &amp; CI](level-4/03-testing-at-scale-ci.md)

**Q129.** What should CI run for PHP?

**Answer:** Linters, static analysis (PHPStan/Psalm) and PHPUnit

Gate merges.

**Q130.** What does PHPStan do?

**Answer:** Finds bugs via static analysis without running code

Levels.

**Q131.** How speed CI?

**Answer:** Cache Composer downloads and parallelize test suites

Efficiency.

**Q132.** What should fail the build?

**Answer:** Failing tests or analysis errors

Quality.

### [04 · Deployment (Docker for PHP-FPM + Nginx)](level-4/04-deployment-docker.md)

**Q133.** Why multi-stage Docker builds for PHP?

**Answer:** Install dev dependencies in one stage, ship a slim runtime image

Smaller images.

**Q134.** What runs PHP in production containers?

**Answer:** PHP-FPM behind Nginx

Standard.

**Q135.** Why disable display_errors in prod?

**Answer:** Prevent leaking internals to users

Log instead.

**Q136.** What's OPcache validate_timestamps=0 for?

**Answer:** Faster production by not checking file changes (restart on deploy)

Config.

### [05 · Performance Optimization at Scale](level-4/05-performance-at-scale.md)

**Q137.** What helps at scale?

**Answer:** Caching, read replicas, queues and horizontal scaling

Architecture.

**Q138.** What's a common bottleneck?

**Answer:** The database (N+1, missing indexes)

Profile.

**Q139.** What does PHP-FPM tuning involve?

**Answer:** Worker counts sized to memory and traffic

pm.max_children.

**Q140.** What's a good performance habit?

**Answer:** Measure with realistic load before and after

Evidence.

### [06 · API Versioning &amp; Documentation](level-4/06-api-versioning-docs.md)

**Q141.** How document an API?

**Answer:** OpenAPI/Swagger generated and validated in CI

Contract.

**Q142.** How deprecate a version?

**Answer:** Announce, add Sunset/Deprecation headers and monitor usage

Communication.

**Q143.** What's backward compatible?

**Answer:** Adding optional fields

Tolerant readers.

**Q144.** What detects who uses v1?

**Answer:** Request metrics by version/client

Evidence.

### [07 · Background Job Processing](level-4/07-background-jobs.md)

**Q145.** What do background jobs need?

**Answer:** Retries with backoff, idempotency and monitoring

Reliable.

**Q146.** What's a scheduler (cron) used for?

**Answer:** Time-based tasks like reports and cleanup

Single runner.

**Q147.** What prevents overlapping runs?

**Answer:** Locks (e.g. database/Redis locks)

withoutOverlapping.

**Q148.** How monitor jobs?

**Answer:** Failure alerts and queue depth metrics

Observability.

### [08 · Working with Microservices in PHP](level-4/08-microservices.md)

**Q149.** When consider microservices for a PHP app?

**Answer:** When teams/scale need independent deployment, not by default

Complexity.

**Q150.** How communicate between services?

**Answer:** HTTP/gRPC for sync, queues/events for async

Contracts.

**Q151.** What does each service own?

**Answer:** Its data and API

Autonomy.

**Q152.** What's a risk?

**Answer:** Distributed failures and operational overhead

Plan for it.

### [09 · Code Quality Tools](level-4/09-code-quality-tools.md)

**Q153.** Which tools enforce style?

**Answer:** PHP-CS-Fixer/PHPCS to PSR-12

Consistency.

**Q154.** What does Rector do?

**Answer:** Automated refactoring and upgrades

Modernize.

**Q155.** What does static analysis catch?

**Answer:** Type errors and dead code before runtime

Quality.

**Q156.** What's a good review practice?

**Answer:** Small PRs with automated checks

Flow.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone combine?

**Answer:** Architecture, security, tests, CI, Docker and monitoring

Production app.

**Q158.** How prove readiness?

**Answer:** Load tests, security review, CI gates and rollback plan

Evidence.

**Q159.** What documentation helps?

**Answer:** README, architecture and runbooks

Onboarding.

**Q160.** What sustains it?

**Answer:** Dependency updates, monitoring and ownership

Maintenance.

## Mixed questions across all levels

**Q161.** What does `==` do with "5" and 5 compared to `===`?

**Answer:** == coerces types and is true; === also checks type and is false

Prefer ===.

**Q162.** What does this print?

```php
$a = ["x" => null];
var_dump(isset($a["x"]), array_key_exists("x", $a), $a["y"] ?? "d");
```

**Answer:** bool(false)
bool(true)
string(1) "d"

isset treats null as unset. Verified by compiling and running it.

**Q163.** What does this print?

```php
function f(int ...$n) { return array_sum($n); }
$args = [1, 2, 3];
echo f(...$args), " ", f(), " ", (fn($x) => $x ** 2)(5);
```

**Answer:** 6 0 25

Spread and variadics. Verified by compiling and running it.

**Q164.** What does this print?

```php
$n = 7;
echo match(true) { $n % 2 === 0 => "even", $n > 5 => "big odd", default => "odd" };
try { echo match($n) { 1 => "a" }; } catch (\UnhandledMatchError $e) { echo " ", get_class($e); }
```

**Answer:** big odd UnhandledMatchError

match is exhaustive. Verified by compiling and running it.

**Q165.** What does this print?

```php
class P { public function __construct(public readonly int $x = 1) {} }
$p = new P(5);
try { $p->x = 9; } catch (Error $e) { echo get_class($e), " ", $e->getMessage(); }
```

**Answer:** Error Cannot modify readonly property P::$x

Readonly properties are immutable. Verified by compiling and running it.

**Q166.** What does this print?

```php
class A { public static function make() { return new static; } public function who() { return static::class; } }
class B extends A {}
echo (B::make())->who(), " ", (new A)->who();
```

**Answer:** B A

Late static binding. Verified by compiling and running it.

**Q167.** When prefer an interface to an abstract class?

**Answer:** To define a contract that unrelated classes can implement

Composition.

**Q168.** What does this print?

```php
function f() { try { return "try"; } finally { echo "[cleanup]"; } }
echo f();
```

**Answer:** [cleanup]try

finally runs before the value is returned. Verified by compiling and running it.

**Q169.** Why are prepared statements the main defence against SQL injection?

**Answer:** User input is sent as data, never parsed as SQL

Bind parameters.

**Q170.** What does this print?

```php
$pdo = new PDO("sqlite::memory:");
$pdo->exec("CREATE TABLE t(id INTEGER PRIMARY KEY, n TEXT)");
$s = $pdo->prepare("INSERT INTO t(n) VALUES(:n)");
$s->execute([":n" => "a"]); $s->execute([":n" => "b"]);
echo $pdo->lastInsertId(), " ", json_encode($pdo->query("SELECT * FROM t")->fetchAll(PDO::FETCH_ASSOC));
```

**Answer:** 2 [{"id":1,"n":"a"},{"id":2,"n":"b"}]

Named parameters and fetch modes. Verified by compiling and running it.

**Q171.** What should happen to the session id after login?

**Answer:** Regenerate it with session_regenerate_id(true)

Prevents session fixation.

**Q172.** Which function stores a password?

**Answer:** password_hash(), verified with password_verify()

Adaptive hashing.

**Q173.** What does this print?

```php
echo htmlspecialchars("<b>\"Tom\" & 'Jerry'</b>"), " ", strlen("日本"), " ", mb_strlen("日本");
```

**Answer:** &amp;lt;b&amp;gt;&amp;quot;Tom&amp;quot; &amp;amp; &amp;#039;Jerry&amp;#039;&amp;lt;/b&amp;gt; 6 2

Escape output; strlen counts bytes. Verified by compiling and running it.

**Q174.** What does PSR-4 map?

**Answer:** Namespace prefixes to directories so classes autoload

Composer autoload.

**Q175.** What do you commit for an application's Composer dependencies?

**Answer:** composer.lock, so every environment installs the same versions

Reproducible builds.

**Q176.** A test needs a repository without a database. What is a good approach?

**Answer:** Inject an in-memory fake implementing the repository interface

Dependency injection.

**Q177.** Why queue slow work such as sending email?

**Answer:** Keeps requests fast and allows retries outside the request

Handlers must be idempotent.

**Q178.** The app is slow under load. What first?

**Answer:** Profile to find the real hotspot (often N+1 queries or missing indexes)

Measure first.

## Practical coding questions

### A shopping cart with discounts

Implement Cart: add (merging quantities and rejecting non-positive quantities with InvalidArgumentException), remove, subtotal, a 10% discount when the subtotal is at least 100.00, total and count. Work in integer cents to avoid float errors.

Starter code:

```
final class Cart {
    private array $lines = [];
    public function add(string $sku, int $qty, int $priceCents): void {
        // throw InvalidArgumentException("qty must be positive") when $qty <= 0
        // merge quantities when the same SKU is added again
    }
    public function remove(string $sku): void { }
    public function subtotal(): int { return 0; }          // sum of qty * price in cents
    public function discount(): int { return 0; }          // 10% (integer cents) when subtotal >= 10000
    public function total(): int { return 0; }
    public function count(): int { return 0; }            // total number of items
}
function money(int $cents): string { return number_format($cents / 100, 2); }

$c = new Cart();
$c->add("pen", 3, 250);
$c->add("book", 2, 4500);
$c->add("pen", 2, 250);
echo "items: ", $c->count(), "\n";
echo "subtotal: ", money($c->subtotal()), "\n";
echo "discount: ", money($c->discount()), "\n";
echo "total: ", money($c->total()), "\n";
$c->remove("book");
echo "after remove: ", money($c->total()), "\n";
try { $c->add("x", 0, 100); } catch (InvalidArgumentException $e) { echo "error: ", $e->getMessage(), "\n"; }
```

Expected output:

```text
items: 7
subtotal: 102.50
discount: 10.25
total: 92.25
after remove: 12.50
error: qty must be positive
```

??? success "Show a solution"

    ```
    final class Cart {
        private array $lines = [];
        public function add(string $sku, int $qty, int $priceCents): void {
            if ($qty <= 0) throw new InvalidArgumentException("qty must be positive");
            if (isset($this->lines[$sku])) { $this->lines[$sku]['qty'] += $qty; }
            else { $this->lines[$sku] = ['qty' => $qty, 'price' => $priceCents]; }
        }
        public function remove(string $sku): void { unset($this->lines[$sku]); }
        public function subtotal(): int {
            return array_sum(array_map(fn($l) => $l['qty'] * $l['price'], $this->lines));
        }
        public function discount(): int {
            $s = $this->subtotal();
            return $s >= 10000 ? intdiv($s * 10, 100) : 0;
        }
        public function total(): int { return $this->subtotal() - $this->discount(); }
        public function count(): int { return array_sum(array_column($this->lines, 'qty')); }
    }
    function money(int $cents): string { return number_format($cents / 100, 2); }

    $c = new Cart();
    $c->add("pen", 3, 250);
    $c->add("book", 2, 4500);
    $c->add("pen", 2, 250);
    echo "items: ", $c->count(), "\n";
    echo "subtotal: ", money($c->subtotal()), "\n";
    echo "discount: ", money($c->discount()), "\n";
    echo "total: ", money($c->total()), "\n";
    $c->remove("book");
    echo "after remove: ", money($c->total()), "\n";
    try { $c->add("x", 0, 100); } catch (InvalidArgumentException $e) { echo "error: ", $e->getMessage(), "\n"; }
    ```

    Money is kept in integer cents so arithmetic is exact; array_sum/array_map compute totals and unset removes a line. Verified by running the solution with PHP.

### A router with a middleware pipeline

Implement Router::dispatch so a request passes through the registered middleware (first registered runs outermost) before reaching the route handler, returning 404 for unknown paths. Auth short-circuits with 401 when the token is wrong, and Logger records the request and response status.

Starter code:

```
interface Middleware { public function handle(array $req, callable $next): array; }
final class Auth implements Middleware {
    public function handle(array $req, callable $next): array {
        if (($req['token'] ?? '') !== 'secret') return ['status' => 401, 'body' => 'unauthorized'];
        return $next($req);
    }
}
final class Logger implements Middleware {
    public static array $log = [];
    public function handle(array $req, callable $next): array {
        self::$log[] = "-> {$req['path']}";
        $res = $next($req);
        self::$log[] = "<- {$res['status']}";
        return $res;
    }
}
final class Router {
    private array $routes = [];
    private array $mw = [];
    public function add(string $path, callable $h): void { $this->routes[$path] = $h; }
    public function use(Middleware $m): void { $this->mw[] = $m; }
    public function dispatch(array $req): array {
        // core handler: 200 + handler output, or 404 "not found"
        // wrap it with the middleware so the FIRST registered middleware runs outermost (array_reduce over the reversed list)
        return ['status' => 500, 'body' => 'todo'];
    }
}

$r = new Router();
$r->use(new Logger());
$r->use(new Auth());
$r->add("/hello", fn($req) => "hello " . ($req['name'] ?? "world"));
foreach ([
    ['path' => '/hello', 'token' => 'secret', 'name' => 'Ada'],
    ['path' => '/hello', 'token' => 'bad'],
    ['path' => '/missing', 'token' => 'secret'],
] as $req) {
    $res = $r->dispatch($req);
    echo $res['status'], " ", $res['body'], "\n";
}
echo implode(" | ", Logger::$log), "\n";
```

Expected output:

```text
200 hello Ada
401 unauthorized
404 not found
-> /hello | <- 200 | -> /hello | <- 401 | -> /missing | <- 404
```

??? success "Show a solution"

    ```
    interface Middleware { public function handle(array $req, callable $next): array; }
    final class Auth implements Middleware {
        public function handle(array $req, callable $next): array {
            if (($req['token'] ?? '') !== 'secret') return ['status' => 401, 'body' => 'unauthorized'];
            return $next($req);
        }
    }
    final class Logger implements Middleware {
        public static array $log = [];
        public function handle(array $req, callable $next): array {
            self::$log[] = "-> {$req['path']}";
            $res = $next($req);
            self::$log[] = "<- {$res['status']}";
            return $res;
        }
    }
    final class Router {
        private array $routes = [];
        private array $mw = [];
        public function add(string $path, callable $h): void { $this->routes[$path] = $h; }
        public function use(Middleware $m): void { $this->mw[] = $m; }
        public function dispatch(array $req): array {
            $core = function (array $r): array {
                $h = $this->routes[$r['path']] ?? null;
                return $h ? ['status' => 200, 'body' => $h($r)] : ['status' => 404, 'body' => 'not found'];
            };
            $chain = array_reduce(array_reverse($this->mw), fn($next, $m) => fn($r) => $m->handle($r, $next), $core);
            return $chain($req);
        }
    }

    $r = new Router();
    $r->use(new Logger());
    $r->use(new Auth());
    $r->add("/hello", fn($req) => "hello " . ($req['name'] ?? "world"));
    foreach ([
        ['path' => '/hello', 'token' => 'secret', 'name' => 'Ada'],
        ['path' => '/hello', 'token' => 'bad'],
        ['path' => '/missing', 'token' => 'secret'],
    ] as $req) {
        $res = $r->dispatch($req);
        echo $res['status'], " ", $res['body'], "\n";
    }
    echo implode(" | ", Logger::$log), "\n";
    ```

    array_reduce over the reversed middleware list builds nested closures (the onion model); a middleware can return early without calling $next. Verified by running the solution with PHP.

---

More free practice: the [PHP Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
