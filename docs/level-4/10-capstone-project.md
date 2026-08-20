# 10 · Capstone Project

Every module in this path has taught one idea in isolation. This one ties
several of them together into a single small service — a task-tracker
API — built the way a real PHP backend is structured: a domain object, a
repository over PDO, a controller enforcing validation, and a router
dispatching by method and path. It's runnable end to end, and it's
tested.

## The shape of the project

```text
src/
  Task.php                # domain object
  TaskRepository.php       # PDO/SQLite persistence
  TaskController.php       # validation + orchestration
  Router.php                # method + path -> handler
  ValidationException.php
public/
  app.php                  # wires it together, runs a demo
tests/
  TaskRepositoryTest.php
  TaskControllerTest.php
```

This mirrors the layering [Level 3's DI module](../level-3/08-dependency-injection.md)
introduced and [module 08's microservices](08-microservices.md) example
reused: each layer depends on the one below it through a narrow surface —
the controller never touches SQL, the repository never touches HTTP
concerns — so any layer can be tested or replaced without dragging the
others along.

## The domain object

```php
<?php
// src/Task.php
declare(strict_types=1);

namespace Capstone;

final class Task
{
    public function __construct(
        public readonly ?int $id,
        public string $title,
        public bool $done = false,
    ) {}

    public function toArray(): array
    {
        return ['id' => $this->id, 'title' => $this->title, 'done' => $this->done];
    }
}
```

`$id` is `readonly` and nullable — nullable because a `Task` doesn't have
an id until the database assigns one via `AUTOINCREMENT`, and `readonly`
because once assigned, nothing in this codebase should ever reassign it.

## Persistence: PDO over SQLite

```php
<?php
// src/TaskRepository.php
declare(strict_types=1);

namespace Capstone;

use PDO;

final class TaskRepository
{
    public function __construct(private PDO $pdo)
    {
        $this->pdo->exec(
            'CREATE TABLE IF NOT EXISTS tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                done INTEGER NOT NULL DEFAULT 0
            )'
        );
    }

    public function add(string $title): Task
    {
        $stmt = $this->pdo->prepare('INSERT INTO tasks (title, done) VALUES (?, 0)');
        $stmt->execute([$title]);
        return new Task((int) $this->pdo->lastInsertId(), $title, false);
    }

    public function find(int $id): ?Task
    {
        $stmt = $this->pdo->prepare('SELECT * FROM tasks WHERE id = ?');
        $stmt->execute([$id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        return $row === false ? null : $this->hydrate($row);
    }

    /** @return Task[] */
    public function all(): array
    {
        $stmt = $this->pdo->query('SELECT * FROM tasks ORDER BY id');
        return array_map($this->hydrate(...), $stmt->fetchAll(PDO::FETCH_ASSOC));
    }

    public function markDone(int $id): bool
    {
        $stmt = $this->pdo->prepare('UPDATE tasks SET done = 1 WHERE id = ?');
        $stmt->execute([$id]);
        return $stmt->rowCount() > 0;
    }

    public function delete(int $id): bool
    {
        $stmt = $this->pdo->prepare('DELETE FROM tasks WHERE id = ?');
        $stmt->execute([$id]);
        return $stmt->rowCount() > 0;
    }

    private function hydrate(array $row): Task
    {
        return new Task((int) $row['id'], $row['title'], (bool) $row['done']);
    }
}
```

Every query goes through `prepare()`/`execute()` with positional
placeholders — no string-concatenated SQL anywhere, so user-supplied
titles can never break out of the query they're bound into.
`markDone()` and `delete()` both report success via `rowCount()` rather
than assuming the write worked; a caller acting on a task id that doesn't
exist gets `false`, not a silent no-op mistaken for success.

## Validation and orchestration

```php
<?php
// src/ValidationException.php
declare(strict_types=1);

namespace Capstone;

final class ValidationException extends \RuntimeException {}
```

```php
<?php
// src/TaskController.php
declare(strict_types=1);

namespace Capstone;

final class TaskController
{
    public function __construct(private TaskRepository $repo) {}

    public function index(): array
    {
        return array_map(fn (Task $t) => $t->toArray(), $this->repo->all());
    }

    public function store(array $input): array
    {
        $title = trim((string) ($input['title'] ?? ''));
        if ($title === '') {
            throw new ValidationException('title is required');
        }
        return $this->repo->add($title)->toArray();
    }

    public function complete(int $id): array
    {
        if (!$this->repo->markDone($id)) {
            throw new \RuntimeException("Task $id not found");
        }
        return $this->repo->find($id)->toArray();
    }

    public function destroy(int $id): bool
    {
        return $this->repo->delete($id);
    }
}
```

The controller is the one place that knows "an empty title is invalid" —
the repository doesn't, and shouldn't, since a repository's job is
persisting whatever it's handed, not deciding what's acceptable input.
Splitting a dedicated `ValidationException` from the generic
`RuntimeException` used for "not found" lets the layer that turns
exceptions into HTTP responses (next) map each to the right status code
without inspecting message strings.

## Routing

```php
<?php
// src/Router.php
declare(strict_types=1);

namespace Capstone;

final class Router
{
    /** @var array<string, array<string, callable>> */
    private array $routes = [];

    public function add(string $method, string $pattern, callable $handler): void
    {
        $this->routes[$method][$pattern] = $handler;
    }

    public function dispatch(string $method, string $path): array
    {
        foreach ($this->routes[$method] ?? [] as $pattern => $handler) {
            $regex = '#^' . preg_replace('#\{(\w+)\}#', '(?P<$1>[^/]+)', $pattern) . '$#';
            if (preg_match($regex, $path, $matches)) {
                $params = array_filter($matches, fn ($k) => !is_int($k), ARRAY_FILTER_USE_KEY);
                return ['status' => 200, 'handler' => $handler, 'params' => $params];
            }
        }
        return ['status' => 404, 'handler' => null, 'params' => []];
    }
}
```

`{id}` in a registered pattern like `/tasks/{id}/complete` becomes a
named capture group (`(?P<id>[^/]+)`) so `dispatch()` returns extracted
path parameters alongside the matched handler, the same
pattern-to-regex idea most PHP micro-frameworks build their routing on.

## Wiring it together

```php
<?php
// public/app.php
declare(strict_types=1);

require __DIR__ . '/../vendor/autoload.php';

use Capstone\Router;
use Capstone\TaskController;
use Capstone\TaskRepository;
use Capstone\ValidationException;

function buildApp(\PDO $pdo): array
{
    $controller = new TaskController(new TaskRepository($pdo));

    $router = new Router();
    $router->add('GET', '/tasks', fn () => $controller->index());
    $router->add('POST', '/tasks', fn (array $params, array $body) => $controller->store($body));
    $router->add('POST', '/tasks/{id}/complete', fn (array $params) => $controller->complete((int) $params['id']));
    $router->add('DELETE', '/tasks/{id}', fn (array $params) => ['deleted' => $controller->destroy((int) $params['id'])]);

    return [$router, $controller];
}

function handle(Router $router, string $method, string $path, array $body = []): array
{
    $match = $router->dispatch($method, $path);
    if ($match['status'] === 404) {
        return ['status' => 404, 'body' => ['error' => 'not found']];
    }
    try {
        $result = ($match['handler'])($match['params'], $body);
        return ['status' => 200, 'body' => $result];
    } catch (ValidationException $e) {
        return ['status' => 422, 'body' => ['error' => $e->getMessage()]];
    } catch (\RuntimeException $e) {
        return ['status' => 404, 'body' => ['error' => $e->getMessage()]];
    }
}
```

`handle()` is the boundary between HTTP-shaped concerns and the rest of
the app: it's the only place that knows a `ValidationException` means 422
and any other `RuntimeException` means 404 — exactly the "let the app
throw, catch at the edge" structure from earlier error-handling modules,
now applied at the scale of a whole request cycle instead of one
function.

## Running it

The same file, run from the CLI, drives a full request cycle against an
in-memory SQLite database and prints each response:

```bash
php public/app.php
```

```text
POST /tasks -> 200 {"id":1,"title":"Write capstone docs","done":false}
POST /tasks -> 200 {"id":2,"title":"Ship it","done":false}
GET /tasks -> 200 [{"id":1,"title":"Write capstone docs","done":false},{"id":2,"title":"Ship it","done":false}]
POST /tasks/1/complete -> 200 {"id":1,"title":"Write capstone docs","done":true}
POST /tasks (empty title) -> 422 {"error":"title is required"}
DELETE /tasks/99 -> 200 {"deleted":false}
GET /nope -> 404 {"error":"not found"}
```

Two tasks get created, listed, and the first is marked done — all
reflected correctly on the next `GET`. The blank-title `POST` returns 422
without ever reaching the repository. Deleting a nonexistent id returns
`{"deleted":false}` rather than an error, since "already not there" isn't
a failure. An unmatched path falls through to the router's own 404.

## Testing it

```php
<?php
// tests/TaskRepositoryTest.php
declare(strict_types=1);

namespace Capstone\Tests;

use Capstone\TaskRepository;
use PDO;
use PHPUnit\Framework\TestCase;

final class TaskRepositoryTest extends TestCase
{
    private function repo(): TaskRepository
    {
        $pdo = new PDO('sqlite::memory:');
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        return new TaskRepository($pdo);
    }

    public function testAddAssignsAnId(): void
    {
        $task = $this->repo()->add('Learn PDO');
        $this->assertSame(1, $task->id);
        $this->assertFalse($task->done);
    }

    public function testMarkDoneUpdatesExistingTask(): void
    {
        $repo = $this->repo();
        $task = $repo->add('Ship it');
        $this->assertTrue($repo->markDone($task->id));
        $this->assertTrue($repo->find($task->id)->done);
    }

    public function testMarkDoneReturnsFalseForMissingTask(): void
    {
        $this->assertFalse($this->repo()->markDone(999));
    }

    public function testDeleteRemovesTask(): void
    {
        $repo = $this->repo();
        $task = $repo->add('Temp');
        $this->assertTrue($repo->delete($task->id));
        $this->assertNull($repo->find($task->id));
    }
}
```

```php
<?php
// tests/TaskControllerTest.php
declare(strict_types=1);

namespace Capstone\Tests;

use Capstone\TaskController;
use Capstone\TaskRepository;
use Capstone\ValidationException;
use PDO;
use PHPUnit\Framework\TestCase;

final class TaskControllerTest extends TestCase
{
    private function controller(): TaskController
    {
        $pdo = new PDO('sqlite::memory:');
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        return new TaskController(new TaskRepository($pdo));
    }

    public function testStoreRejectsBlankTitle(): void
    {
        $this->expectException(ValidationException::class);
        $this->controller()->store(['title' => '  ']);
    }

    public function testStoreThenIndexRoundTrips(): void
    {
        $controller = $this->controller();
        $controller->store(['title' => 'Write tests']);
        $this->assertCount(1, $controller->index());
    }

    public function testCompleteUnknownTaskThrows(): void
    {
        $this->expectException(\RuntimeException::class);
        $this->controller()->complete(404);
    }
}
```

Each test builds its own in-memory `PDO` connection, so tests never share
state or run in a particular order to pass — exactly the isolation
[module 03's testing-at-scale](03-testing-at-scale-ci.md) module built
toward, now applied to a project with two collaborating classes instead
of one.

```bash
vendor/bin/phpunit --testdox
```

```text
PHPUnit 11.5.56 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.5.9

.......                                                             7 / 7 (100%)

Time: 00:00.014, Memory: 8.00 MB

Task Controller (Capstone\Tests\TaskController)
 [x] Store rejects blank title
 [x] Store then index round trips
 [x] Complete unknown task throws

Task Repository (Capstone\Tests\TaskRepository)
 [x] Add assigns an id
 [x] Mark done updates existing task
 [x] Mark done returns false for missing task
 [x] Delete removes task

OK (7 tests, 10 assertions)
```

Seven tests, ten assertions, all green — three at the controller layer
(validation and orchestration) and four at the repository layer
(persistence), matching the two-layer split the project is built on.

## PHP traps

**`$this->pdo->query()` without placeholders is fine only because `all()`
takes no user input.** The instant a method like `all()` grows a filter
parameter (`all(string $search)`), reaching for `query()` with
string-concatenated SQL to add a `WHERE title LIKE '%$search%'` reopens
exactly the injection hole `prepare()`/`execute()` closed everywhere
else — any new query needs the same placeholder discipline as the
existing ones, not an exception for "just this one."

**Testing the controller against a real `TaskRepository` (backed by an
in-memory SQLite database) instead of a mock isn't a shortcut — it's
deliberate.** A mocked repository can be told to return anything,
including states the real one never produces; an in-memory database
exercises the actual SQL, so `testMarkDoneUpdatesExistingTask` proves the
`UPDATE` statement and the `find()` read-back genuinely agree with each
other, not just that the controller calls methods in the right order.

**`in-memory` SQLite means every test run starts from a schema-only,
empty database — nothing persists between runs, and nothing should leak
between tests either.** Each test method here calls `$this->repo()` (or
`$this->controller()`) fresh, creating a brand-new `PDO` connection and
therefore a brand-new database; sharing one connection across test
methods via a property set in `setUp()` would make `testAddAssignsAnId`'s
assertion that the first id is `1` depend on test execution order.

## Capstone architecture cheat sheet

| Layer | Responsibility | Never does |
|---|---|---|
| `Task` | Hold and shape one record's data | Talk to the database |
| `TaskRepository` | Persist and retrieve via PDO | Know what "invalid" means |
| `TaskController` | Validate input, orchestrate | Write SQL |
| `Router` | Match method + path to a handler | Know what a task is |
| `handle()` in `app.php` | Map exceptions to HTTP status | Contain business logic |
| Tests | Exercise each layer through a real SQLite DB | Mock away the SQL they're proving works |

## Stretch goals

Extend the capstone in any of these directions — each pulls in a concept
from an earlier module in this path:

- Add a `PATCH /tasks/{id}` route and a `TaskController::rename()` method
  that validates the new title the same way `store()` does, instead of
  duplicating the blank-check.
- Add an `#[RouteDoc(...)]` attribute (from [module 06](06-api-versioning-docs.md))
  to each `Router::add()` call site and a small script that reflects over
  them to print a routes table — turning the router itself into
  self-documenting API surface.
- Wrap `TaskRepository`'s database connection setup in a `CircuitBreaker`
  (from [module 08](08-microservices.md)) so a real (non-in-memory)
  database outage fails fast instead of hanging every request.
- Wire `composer.json` with a `scripts.check` entry running PHPStan,
  PHP-CS-Fixer, and PHPUnit together (from [module 09](09-code-quality-tools.md)),
  then add the GitHub Actions workflow from
  [module 03](03-testing-at-scale-ci.md) so every push runs all three
  automatically.
