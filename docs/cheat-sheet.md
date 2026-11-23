---
title: "PHP Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free PHP cheat sheet: 35 quick-reference tables from every level of the PHP Mastery Path, each linked to its full lesson."
---

# PHP Cheat Sheet

Quick-reference tables from **35 lessons** of the free PHP Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

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

### [03 · Control Flow](level-1/03-control-flow.md)

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

### [04 · Functions](level-1/04-functions.md)

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

### [05 · Arrays](level-1/05-arrays.md)

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

### [06 · Strings &amp; String Functions](level-1/06-strings.md)

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

### [07 · Classes &amp; Objects Basics](level-1/07-classes-objects.md)

| Concept | Meaning |
|---------|---------|
| Class | Blueprint / type definition |
| Object (instance) | A concrete value created with `new` |
| Property | A variable that lives on each instance (or on the class, if `static`) |
| Constructor (`__construct`) | Special method that initializes a new instance |
| `$this` | Reference to the current instance |
| `self::` | Reference to the current class (for static members) |
| `public` / `protected` / `private` | Visibility, from most to least accessible |
| `->` | Access a property/method on an instance |
| `::` | Access a static member, class constant, or parent method |

### [08 · Error Handling Basics](level-1/08-error-handling.md)

| Keyword / class | Purpose |
|------------------|---------|
| `try { ... }` | Wrap code that might throw |
| `catch (Type $e) { ... }` | Handle a specific exception/error type |
| `catch (TypeA \| TypeB $e) { ... }` | Handle either of two types the same way |
| `finally { ... }` | Always runs, for cleanup |
| `throw new Exception("msg")` | Raise an exception |
| `$e->getMessage()` | Retrieve the error message |
| `$e->getCode()` | Retrieve an optional numeric error code |
| `Throwable` | Interface implemented by both `Exception` and `Error` |

### [09 · Composer &amp; Package Basics](level-1/09-composer-basics.md)

| Command | Purpose |
|---------|---------|
| `composer init` | Interactively create a new `composer.json` |
| `composer require <pkg>` | Add and install a production dependency |
| `composer require --dev <pkg>` | Add and install a dev-only dependency |
| `composer install` | Install exact versions from `composer.lock` |
| `composer update` | Resolve and install the newest versions allowed by constraints |
| `composer dump-autoload` | Regenerate the autoloader (e.g. after adding a new class) |
| `composer show` | List installed packages and their versions |

## Level 2 · Intermediate

### [02 · Error Handling Advanced](level-2/02-error-handling-advanced.md)

| Tool | Purpose |
|------|---------|
| `abstract class AppException extends RuntimeException` | Root of a custom exception hierarchy |
| `new Foo($msg, previous: $e)` | Chain the original exception so context isn't lost |
| `$e->getPrevious()` | Retrieve the chained (wrapped) exception, or `null` |
| `set_error_handler()` | Convert PHP warnings/notices into catchable `ErrorException`s |
| `set_exception_handler()` | Last-resort handler for anything left uncaught |
| `restore_error_handler()` | Undo a custom `set_error_handler()` registration |
| `finally` | Runs on every path out of `try`/`catch` — cleanup only, don't `return` here |

### [03 · Working with Databases (PDO)](level-2/03-databases-pdo.md)

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

### [04 · Sessions &amp; Forms](level-2/04-sessions-forms.md)

| Tool | Purpose |
|------|---------|
| `$_GET`, `$_POST` | Read query-string vs. request-body input |
| `htmlspecialchars($value)` | Escape output to prevent XSS |
| `filter_var($v, FILTER_VALIDATE_EMAIL)` | Validate common formats |
| `session_start()` | Begin/resume a session (call before any output) |
| `$_SESSION` | Server-side, per-visitor storage across requests |
| `session_regenerate_id()` | Issue a fresh session ID (call on login) |
| `password_hash()` / `password_verify()` | Safely store and check passwords |
| `hash_equals()` | Timing-safe string comparison, e.g. for CSRF tokens |

### [05 · Testing with PHPUnit](level-2/05-phpunit-testing.md)

| Command / concept | Purpose |
|--------------------|---------|
| `vendor/bin/phpunit tests` | Run every test in the `tests/` directory |
| `vendor/bin/phpunit --filter testName` | Run only tests matching a name |
| `setUp()` / `tearDown()` | Runs before / after each test method |
| `#[DataProvider("method")]` | Run one test body across many input rows |
| `$this->createMock(Interface::class)` | Create a fake collaborator for isolated testing |
| `phpunit.xml` | Config file: test suite locations, bootstrap file, coverage settings |

### [06 · Working with JSON/APIs](level-2/06-json-apis.md)

| Function / flag | Purpose |
|------------------|---------|
| `json_encode($value, $flags)` | PHP value → JSON string |
| `json_decode($json, true, flags: ...)` | JSON string → associative array |
| `json_decode($json)` (no `true`) | JSON string → `stdClass` object(s) |
| `JSON_THROW_ON_ERROR` | Throw `JsonException` instead of failing silently |
| `JSON_PRETTY_PRINT` | Human-readable, indented output |
| `JsonSerializable` | Interface to control exactly how an object encodes |
| `CURLOPT_RETURNTRANSFER` | Return the response body instead of printing it |
| `http_response_code()` | Set the HTTP status code of your own response |

### [07 · Autoloading &amp; PSR Standards](level-2/07-autoloading-psr.md)

| Concept | Summary |
|---------|---------|
| `spl_autoload_register()` | Low-level function Composer's autoloader is built on |
| PSR-4 rule | `Namespace\Prefix\ClassName` → `mapped/dir/ClassName.php` |
| One class per file | Required for PSR-4's mechanical path computation to work |
| `classmap` | Scans and indexes classes that don't follow PSR-4 |
| `files` | Always-loaded plain function files |
| `composer dump-autoload` | Regenerate the autoloader after structural changes |
| `composer dump-autoload -o` | Production-optimized, pre-computed class map |

### [08 · File Uploads &amp; Validation](level-2/08-file-uploads-validation.md)

| Check | Why |
|-------|-----|
| `$_FILES[...]["error"] === UPLOAD_ERR_OK` | Confirms the upload actually completed before touching other fields |
| Your own size check | Enforces a per-form limit, independent of `php.ini` defaults |
| `finfo::file()` on the real bytes | The client's declared `type`/extension can be anything |
| Generate your own filename | Avoids path traversal, null bytes, and overwrite collisions |
| `move_uploaded_file()`, never `rename()` | Verifies the source was a genuine PHP upload |
| Store outside the web root | A validation gap can't become remote code execution |

### [09 · Working with Dates](level-2/09-working-with-dates.md)

| Tool | Purpose |
|------|---------|
| `DateTimeImmutable` | Preferred date/time type — every operation returns a new instance |
| `DateTime` | Mutable equivalent — `modify()`/`add()`/`sub()` change the object in place |
| `->format($fmt)` | Render as a string using format characters |
| `DateTimeImmutable::createFromFormat($fmt, $str)` | Strict parsing of a known input format |
| `DateInterval` | A span of time (`P1M`, `P10D`, `P1W`) used with `add()`/`sub()` |
| `->diff($other)` | Compute the `DateInterval` between two dates |
| `DateTimeZone` | Represents a named time zone (`"UTC"`, `"Asia/Kolkata"`) |
| `->setTimezone($tz)` | Return the same instant, viewed in a different time zone |

## Level 3 · Advanced

### [01 · Building APIs](level-3/01-building-apis.md)

| Code | Meaning | When to use it |
|------|---------|-----------------|
| `200 OK` | Success | Successful `GET`, `PUT`, `PATCH`, `DELETE` |
| `201 Created` | Resource created | Successful `POST` that creates something new |
| `204 No Content` | Success, no body | Successful `DELETE` with nothing to return |
| `400 Bad Request` | Malformed request | Invalid JSON, missing required field |
| `401 Unauthorized` | Not authenticated | Missing or invalid credentials |
| `403 Forbidden` | Authenticated, not allowed | Valid user, insufficient permissions |
| `404 Not Found` | No such resource | Route or record doesn't exist |
| `422 Unprocessable Entity` | Semantically invalid | Well-formed JSON that fails validation rules |
| `500 Internal Server Error` | Server bug | An uncaught exception — never expose the stack trace to the client |

### [02 · Design Patterns in PHP](level-3/02-design-patterns.md)

| Pattern | Problem it solves | PHP-idiomatic hook |
|---------|--------------------|--------------------|
| Singleton | Exactly one shared instance | Private constructor + `??=` in a static method |
| Factory Method | Centralize which concrete class gets built | `match` returning an interface type |
| Strategy | Swap an algorithm without changing the caller | Constructor-injected interface |
| Observer | Notify many listeners without coupling to them | An array of interface instances + `foreach` |
| Decorator | Add behavior by wrapping, not inheriting | Class implementing the same interface it wraps |

### [03 · Security](level-3/03-security.md)

| Risk | Defense | PHP function/setting |
|------|---------|-----------------------|
| SQL injection (values) | Bound parameters | `PDO::prepare()` + `execute([...])` |
| SQL injection (identifiers) | Allowlist, never bind | `in_array($col, $allowed, true)` |
| Stored XSS | Escape on output, not input | `htmlspecialchars()` |
| CSRF | Per-session token, checked on every write | `hash_equals($session, $submitted)` |
| Weak password storage | Slow, salted, adaptive hash | `password_hash()` / `password_verify()` |
| Timing attack on secrets | Constant-time comparison | `hash_equals()` |
| PHP object injection | Never unserialize untrusted bytes | `unserialize($x, ["allowed_classes" => [...]])` or `json_decode()` |
| Clickjacking / MIME sniffing | Restrictive response headers | `header("X-Frame-Options: DENY")` |
| Session/cookie theft | Restrict cookie scope and transport | `session_set_cookie_params([...])` |

### [04 · Testing Advanced (Mocking)](level-3/04-testing-advanced-mocking.md)

| Call | What you get |
|------|--------------|
| `$this->createStub(I::class)` | Canned answers only — never fails a test on its own |
| `$this->createMock(I::class)` | Stub plus verifiable `expects()` expectations |
| `->method("m")->willReturn($v)` | Same answer every call |
| `->willReturnMap([[$a, $r], ...])` | Answer chosen by argument values (last element = return) |
| `->willReturnOnConsecutiveCalls($a, $b)` | A different answer per call, in order |
| `->willReturnCallback($fn)` | Answer computed from the arguments |
| `->willThrowException(new E())` | Simulate the failure path |
| `->expects($this->once())` | Must be called exactly once |
| `->expects($this->never())` | Must never be called |
| `->with($a, $b)` | Constrain the arguments (matchers like `$this->stringContains()` allowed) |
| Hand-written fake | Working implementation with a shortcut — best for stateful collaborators |

### [05 · Performance &amp; Caching](level-3/05-performance-caching.md)

| Tool | Caches | Scope | Survives restart? |
|------|--------|-------|--------------------|
| OPcache | Compiled bytecode | Per server | No (rebuilt on first request) |
| APCu | Arbitrary values | Per server process | No |
| Redis / Memcached | Arbitrary values | Shared across servers | Yes (Redis, if persistence configured) |
| File cache (this module) | Arbitrary values | Per filesystem | Yes |
| `remember($key, $ttl, $fn)` | Pattern: check cache, else compute + store | — | — |

### [06 · Working with Queues](level-3/06-working-with-queues.md)

| Concept | This module's file queue | Redis / RabbitMQ |
|---|---|---|
| Enqueue | `push()` writes a JSON file to `pending/` | `LPUSH` / `basic_publish` |
| Claim a job | `pop()` + atomic `rename()` | `BRPOP` (blocking) / consumer callback |
| Acknowledge success | move to `done/` | `basic_ack` |
| Retry | move back to `pending/`, increment `attempts` | requeue / redelivery on nack |
| Give up | move to `failed/` | dead-letter queue/exchange |
| Worker wakes on new job | polling (must check repeatedly) | blocking pop / push notification |

### [07 · Middleware Patterns](level-3/07-middleware-patterns.md)

| Concept | What it means here |
|---|---|
| `Middleware::handle($request, $next)` | One layer; decides whether/how to call `$next` |
| `$next($request)` | Continue to the next layer (or the controller) |
| Not calling `$next()` | Short-circuit — return a response immediately (auth failure, rate limit) |
| `Pipeline::pipe()` order | First piped = outermost = runs first going in, last coming out |
| `array_reduce` + `array_reverse` | Standard trick for building a nested chain from a flat list |
| Mutating `$request->attributes` | How inner layers see data set by outer layers (e.g. the authenticated user) |

### [08 · Dependency Injection Basics](level-3/08-dependency-injection.md)

| Concept | What it means |
|---|---|
| Constructor injection | Dependencies declared as typed constructor parameters |
| Depend on interfaces | Makes the concrete implementation swappable (real vs. fake) |
| Manual wiring | `new A(new B(), new C())` — fine for a handful of classes |
| Container | Centralizes "how to build X"; resolves dependency graphs |
| `bind($abstract, $factory)` | Explicit recipe for building an interface/class |
| Autowiring | Container inspects constructors via `Reflection` and resolves automatically |
| Service locator (anti-pattern) | Injecting the container itself and calling `->get()` inside a class |

### [09 · Working with Composer Packages at Scale](level-3/09-composer-at-scale.md)

| Command | Purpose |
|---|---|
| `composer install` | Install exact versions from `composer.lock` (reproducible) |
| `composer update [pkg]` | Re-resolve constraints, rewrite `composer.lock` |
| `composer require pkg:^1.0` | Add a dependency and update `composer.json` + lock together |
| `composer require --dev pkg` | Add a dev-only dependency |
| `composer dump-autoload` | Regenerate the autoloader after adding classes manually |
| `composer validate --strict` | Catch `composer.json` schema problems in CI |
| `composer install --no-dev --optimize-autoloader` | Production install: no dev deps, static class map |
| `composer show` | List installed packages and versions |

## Level 4 · Master

### [01 · Advanced Framework Patterns](level-4/01-advanced-framework-patterns.md)

| Pattern | Purpose |
|---|---|
| Service provider | Groups related container bindings + boot logic per feature |
| `register()` | Declare bindings only — no resolving yet |
| `boot()` | Runs after all providers registered — safe to resolve |
| `__get()` / `__set()` | Intercept access to undeclared/inaccessible properties |
| `static::` (late static binding) | Resolves against the calling class, not the defining one |
| `new static()` | Instantiate the actual subclass, not the base class |

### [02 · Security at Scale](level-4/02-security-at-scale.md)

| Concern | Tool | Key rule |
|---|---|---|
| Secrets | `.env` / platform secret store | Never commit; access through one central reader |
| Missing secret | `require()`-style loud failure | Fail at startup, not deep in a request |
| CSRF | Random token per session, checked per state-changing request | `hash_equals()`, never `===` |
| Token/secret comparison | `hash_equals($expected, $actual)` | Constant-time, avoids timing attacks |
| Password storage | `password_hash()` / `password_verify()` | Never store or log plaintext |
| Aging hashes | `password_needs_rehash()` | Check and upgrade on successful login |

### [03 · Testing at Scale &amp; CI](level-4/03-testing-at-scale-ci.md)

| Tool | Purpose |
|---|---|
| `phpunit.xml` | Committed test configuration — bootstrap, suites, group filters |
| `#[Group('slow')]` | Marks a test excludable/selectable by name |
| `<groups><exclude>` | Skips a named group by default |
| `--group slow` | Runs *only* the named group |
| `composer install` in CI | Reproducible install from the committed lock file |
| `--coverage-text` (needs Xdebug/PCOV) | Reports which lines the suite actually exercises |
| Coverage floor (e.g. 80%) | A practical CI gate — catches obviously-untested code without chasing 100% |

### [04 · Deployment (Docker for PHP-FPM + Nginx)](level-4/04-deployment-docker.md)

| Piece | Role |
|---|---|
| PHP-FPM | Process pool that actually executes PHP for each request |
| Nginx | Serves static assets directly; proxies `.php` requests to FPM |
| `fastcgi_pass app:9000` | Nginx forwards dynamic requests to the FPM service |
| Dockerfile layer ordering | Dependencies installed before app code copied — cache-friendly |
| `opcache.validate_timestamps=0` | Production-only: skip per-request file change checks |
| `USER www-data` | Run the PHP process as an unprivileged user, not root |
| `docker compose up --build` | Builds and starts both services on one shared network |

### [05 · Performance Optimization at Scale](level-4/05-performance-at-scale.md)

| Problem | Symptom | Fix |
|---|---|---|
| N+1 queries | 1 query becomes 1+N as a list grows | `JOIN`, or eager-load related data up front |
| Large array in memory | `memory_get_usage()` spikes with dataset size | Generator (`yield`) instead of building an array |
| Unbounded `SELECT *` | Query time and memory both grow with table size | `LIMIT`/`OFFSET` pagination, processed via a generator |
| Slow `JOIN` | Fixing N+1 introduces a new slow query | Check `EXPLAIN`/`EXPLAIN QUERY PLAN`; add indexes on join columns |
| Generator reused | `Cannot rewind a generator` exception | Call the generator function again, or materialize once into an array |

### [06 · API Versioning &amp; Documentation](level-4/06-api-versioning-docs.md)

| Concept | Mechanism | Example |
|---|---|---|
| Header-based versioning | `Accept: application/vnd.*.vN+json` | `V1Response` vs. `V2Response` |
| Version isolation | Each version has its own pure `transform()` | Same `$data`, different shape out |
| Unsupported version | Fail with a clear error, not a silent default | `RuntimeException` in `handle()` |
| Docs as metadata | PHP 8 attributes (`#[RouteDoc(...)]`) | Read via `Reflection` |
| Doc generation | `getAttributes()` + `newInstance()` | `generateDocs()` |
| Real-world equivalent | `zircote/swagger-php`, OpenAPI spec generation | Attributes -> JSON/YAML spec |

### [07 · Background Job Processing](level-4/07-background-jobs.md)

| Concept | Purpose |
|---|---|
| `Job` interface | Each background task is a class: data + `handle()` logic together |
| `dispatch()` | Application-facing entry point; hides sync vs. queued behind one call |
| `runSync()` | Bypass the queue — useful for tests/CLI, risky as a request-path default |
| `payload()` | How a job serializes itself onto a real transport (Redis, `FileQueue`) |
| `ScheduledTask::isDue()` | Interval check driving cron-style recurring work |
| Single cron entry + per-task `isDue()` | One `* * * * *` line can drive many independent schedules |

### [08 · Working with Microservices in PHP](level-4/08-microservices.md)

| Concept | Purpose |
|---|---|
| Service interface (`UserService`) | Caller depends on a contract, not a transport (HTTP vs. in-process) |
| Each service owns its data | No service reaches into another's database directly |
| Circuit breaker | Fails fast after repeated failures instead of piling up slow requests |
| Closed / open / half-open | Normal / failing-fast / probing-for-recovery states |
| Per-dependency breaker | One breaker per downstream service, not shared globally |
| Client-side timeout | What actually makes a hung call throw so the breaker can react |

### [09 · Code Quality Tools](level-4/09-code-quality-tools.md)

| Tool | Question it answers | Command used here |
|---|---|---|
| PHPStan | Do the types actually line up? | `phpstan analyse` |
| PHP-CS-Fixer | Does the code match one consistent style? | `php-cs-fixer fix --dry-run --diff` |
| PHPUnit | Do the behaviors I claim work actually work? | `phpunit --testdox` |
| Coverage report | Which lines did the test suite never reach? | `phpunit --coverage-text` (needs Xdebug/PCOV) |
| Baseline | How to adopt strict analysis on old code | `phpstan analyse --generate-baseline` |
| CI wiring | What makes any of this non-optional | `composer check` run in the pipeline |

### [10 · Capstone Project](level-4/10-capstone-project.md)

| Layer | Responsibility | Never does |
|---|---|---|
| `Task` | Hold and shape one record's data | Talk to the database |
| `TaskRepository` | Persist and retrieve via PDO | Know what "invalid" means |
| `TaskController` | Validate input, orchestrate | Write SQL |
| `Router` | Match method + path to a handler | Know what a task is |
| `handle()` in `app.php` | Map exceptions to HTTP status | Contain business logic |
| Tests | Exercise each layer through a real SQLite DB | Mock away the SQL they're proving works |

---

Go deeper with the full [PHP Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
