# 10 · Project — REST API Service

This project pulls together everything from Level 3: a layered service
(repository → controller), dependency injection through constructors, a
tiny router, and PHPUnit tests against a real database — SQLite via PDO, so
the whole thing runs with nothing installed beyond PHP itself. No framework;
seeing the pieces built by hand makes it obvious what a framework is
automating for you later.

## Project layout

```text
task-api/
├── composer.json
├── demo.php
├── src/
│   ├── Database.php
│   ├── TaskRepository.php
│   ├── TaskController.php
│   └── Router.php
└── tests/
    └── TaskApiTest.php
```

```json
{
    "name": "acme/task-api",
    "description": "REST API for a task list, PDO/SQLite-backed",
    "license": "proprietary",
    "require": {
        "php": "^8.2",
        "ext-pdo": "*"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0"
    },
    "autoload": {
        "psr-4": { "TaskApi\\": "src/" }
    },
    "autoload-dev": {
        "psr-4": { "TaskApi\\Tests\\": "tests/" }
    }
}
```

## The persistence layer

`Database::connect()` opens a PDO connection to SQLite and creates the
schema if it doesn't already exist — pointed at `:memory:`, this needs no
running database server at all, which is what makes the whole project
runnable anywhere PHP is installed.

```php
<?php
// src/Database.php
declare(strict_types=1);

namespace TaskApi;

use PDO;

final class Database
{
    public static function connect(string $path = ':memory:'): PDO
    {
        $pdo = new PDO("sqlite:$path");
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $pdo->exec('
            CREATE TABLE IF NOT EXISTS tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                done INTEGER NOT NULL DEFAULT 0,
                created_at TEXT NOT NULL
            )
        ');
        return $pdo;
    }
}
```

`TaskRepository` is the only class that knows SQL exists — every query uses
a **prepared statement with bound parameters**, never string-interpolated
values, which is what makes it immune to SQL injection regardless of what a
caller passes as a title.

```php
<?php
// src/TaskRepository.php
declare(strict_types=1);

namespace TaskApi;

use PDO;

final class TaskRepository
{
    public function __construct(private PDO $pdo) {}

    public function create(string $title): array
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO tasks (title, done, created_at) VALUES (:title, 0, :created_at)'
        );
        $stmt->execute(['title' => $title, 'created_at' => date('c')]);
        return $this->find((int) $this->pdo->lastInsertId());
    }

    public function find(int $id): ?array
    {
        $stmt = $this->pdo->prepare('SELECT * FROM tasks WHERE id = :id');
        $stmt->execute(['id' => $id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        return $row === false ? null : $this->hydrate($row);
    }

    public function all(): array
    {
        $stmt = $this->pdo->query('SELECT * FROM tasks ORDER BY id');
        return array_map($this->hydrate(...), $stmt->fetchAll(PDO::FETCH_ASSOC));
    }

    public function markDone(int $id): bool
    {
        $stmt = $this->pdo->prepare('UPDATE tasks SET done = 1 WHERE id = :id');
        $stmt->execute(['id' => $id]);
        return $stmt->rowCount() > 0;
    }

    public function delete(int $id): bool
    {
        $stmt = $this->pdo->prepare('DELETE FROM tasks WHERE id = :id');
        $stmt->execute(['id' => $id]);
        return $stmt->rowCount() > 0;
    }

    private function hydrate(array $row): array
    {
        return [
            'id' => (int) $row['id'],
            'title' => $row['title'],
            'done' => (bool) $row['done'],
            'created_at' => $row['created_at'],
        ];
    }
}
```

`markDone()` and `delete()` return `bool` based on `rowCount()` rather than
just running the statement — that's what lets the controller distinguish
"updated a real row" from "that id didn't exist" without a separate lookup.

## The HTTP layer

`TaskController` depends on `TaskRepository` through its constructor (plain
[dependency injection](08-dependency-injection.md) — no container needed at
this scale) and never touches PDO directly. Every method returns
`[$status, $body]`, keeping the controller framework-agnostic: nothing here
assumes `$_GET`, superglobals, or `echo`.

```php
<?php
// src/TaskController.php
declare(strict_types=1);

namespace TaskApi;

final class TaskController
{
    public function __construct(private TaskRepository $repo) {}

    public function index(): array
    {
        return [200, $this->repo->all()];
    }

    public function show(array $params): array
    {
        $task = $this->repo->find((int) $params['id']);
        return $task === null ? [404, ['error' => 'Task not found']] : [200, $task];
    }

    public function store(array $body): array
    {
        $title = trim((string) ($body['title'] ?? ''));
        if ($title === '') {
            return [422, ['error' => 'title is required']];
        }
        return [201, $this->repo->create($title)];
    }

    public function complete(array $params): array
    {
        $ok = $this->repo->markDone((int) $params['id']);
        return $ok ? [200, ['status' => 'done']] : [404, ['error' => 'Task not found']];
    }

    public function destroy(array $params): array
    {
        $ok = $this->repo->delete((int) $params['id']);
        return $ok ? [200, ['status' => 'deleted']] : [404, ['error' => 'Task not found']];
    }
}
```

## A minimal router

Routing itself — mapping `GET /tasks/{id}` to a handler and extracting
`{id}` — is orthogonal to the persistence and controller logic above, so
it's kept as its own class: `{name}` segments in a registered pattern
become a regex capture group.

```php
<?php
// src/Router.php
declare(strict_types=1);

namespace TaskApi;

final class Router
{
    /** @var array<string, array<string, callable>> */
    private array $routes = [];

    public function add(string $method, string $pattern, callable $handler): void
    {
        $this->routes[$method][$pattern] = $handler;
    }

    /** @return array{0:int,1:mixed} [status, body] */
    public function dispatch(string $method, string $path): array
    {
        foreach ($this->routes[$method] ?? [] as $pattern => $handler) {
            $regex = '#^' . preg_replace('#\{(\w+)\}#', '(?P<$1>[^/]+)', $pattern) . '$#';
            if (preg_match($regex, $path, $matches)) {
                $params = array_filter($matches, fn($k) => !is_int($k), ARRAY_FILTER_USE_KEY);
                return $handler($params);
            }
        }
        return [404, ['error' => 'Not Found']];
    }
}
```

```php
<?php
// router_demo.php
require __DIR__ . '/vendor/autoload.php';
use TaskApi\Router;

$router = new Router();
$router->add('GET', '/tasks/{id}', fn($p) => [200, "showing task {$p['id']}"]);

[$status, $body] = $router->dispatch('GET', '/tasks/42');
echo "$status $body\n";

[$status, $body] = $router->dispatch('GET', '/nope');
echo "$status " . json_encode($body) . "\n";
```

```text
200 showing task 42
404 {"error":"Not Found"}
```

## Running it

`demo.php` wires an in-memory SQLite database into a repository, into a
controller, and drives it directly (bypassing the router, since there's no
real HTTP server here) to exercise the whole create/list/update/delete flow.

```php
<?php
// demo.php
declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

use TaskApi\Database;
use TaskApi\TaskController;
use TaskApi\TaskRepository;

$pdo = Database::connect(); // in-memory SQLite -- no server needed
$controller = new TaskController(new TaskRepository($pdo));

function show(string $label, array $result): void
{
    [$status, $body] = $result;
    echo "$label -> $status " . json_encode($body) . "\n";
}

show('POST /tasks {title: "Write module"}', $controller->store(['title' => 'Write module']));
show('POST /tasks {title: "Ship it"}', $controller->store(['title' => 'Ship it']));
show('POST /tasks {title: ""}', $controller->store(['title' => '']));
show('GET /tasks', $controller->index());
show('GET /tasks/1', $controller->show(['id' => '1']));
show('PATCH /tasks/1/complete', $controller->complete(['id' => '1']));
show('GET /tasks/1 (after complete)', $controller->show(['id' => '1']));
show('DELETE /tasks/2', $controller->destroy(['id' => '2']));
show('GET /tasks (after delete)', $controller->index());
show('GET /tasks/999', $controller->show(['id' => '999']));
```

```bash
composer dump-autoload
php demo.php
```

```text
POST /tasks {title: "Write module"} -> 201 {"id":1,"title":"Write module","done":false,"created_at":"2026-08-18T17:05:13+00:00"}
POST /tasks {title: "Ship it"} -> 201 {"id":2,"title":"Ship it","done":false,"created_at":"2026-08-18T17:05:13+00:00"}
POST /tasks {title: ""} -> 422 {"error":"title is required"}
GET /tasks -> 200 [{"id":1,"title":"Write module","done":false,"created_at":"2026-08-18T17:05:13+00:00"},{"id":2,"title":"Ship it","done":false,"created_at":"2026-08-18T17:05:13+00:00"}]
GET /tasks/1 -> 200 {"id":1,"title":"Write module","done":false,"created_at":"2026-08-18T17:05:13+00:00"}
PATCH /tasks/1/complete -> 200 {"status":"done"}
GET /tasks/1 (after complete) -> 200 {"id":1,"title":"Write module","done":true,"created_at":"2026-08-18T17:05:13+00:00"}
DELETE /tasks/2 -> 200 {"status":"deleted"}
GET /tasks (after delete) -> 200 [{"id":1,"title":"Write module","done":true,"created_at":"2026-08-18T17:05:13+00:00"}]
GET /tasks/999 -> 404 {"error":"Task not found"}
```

## Testing against a real database

Each test gets its **own** fresh `:memory:` database via `setUp()` — no
shared state between tests, no cleanup step needed, and no risk of test
order affecting results, because SQLite discards an in-memory database the
moment the connection closes.

```php
<?php
// tests/TaskApiTest.php
declare(strict_types=1);

namespace TaskApi\Tests;

use PHPUnit\Framework\TestCase;
use TaskApi\Database;
use TaskApi\TaskController;
use TaskApi\TaskRepository;

final class TaskApiTest extends TestCase
{
    private TaskController $controller;

    protected function setUp(): void
    {
        $pdo = Database::connect(); // fresh in-memory DB per test
        $this->controller = new TaskController(new TaskRepository($pdo));
    }

    public function testCreatingATaskReturns201WithTheStoredTask(): void
    {
        [$status, $body] = $this->controller->store(['title' => 'Write tests']);

        $this->assertSame(201, $status);
        $this->assertSame('Write tests', $body['title']);
        $this->assertFalse($body['done']);
    }

    public function testCreatingATaskWithBlankTitleReturns422(): void
    {
        [$status, $body] = $this->controller->store(['title' => '   ']);

        $this->assertSame(422, $status);
        $this->assertArrayHasKey('error', $body);
    }

    public function testIndexListsAllCreatedTasks(): void
    {
        $this->controller->store(['title' => 'One']);
        $this->controller->store(['title' => 'Two']);

        [$status, $body] = $this->controller->index();

        $this->assertSame(200, $status);
        $this->assertCount(2, $body);
    }

    public function testShowReturns404ForMissingTask(): void
    {
        [$status, $body] = $this->controller->show(['id' => '999']);

        $this->assertSame(404, $status);
        $this->assertSame('Task not found', $body['error']);
    }

    public function testCompleteMarksTaskDone(): void
    {
        [, $created] = $this->controller->store(['title' => 'Finish it']);

        [$status] = $this->controller->complete(['id' => (string) $created['id']]);
        [, $fetched] = $this->controller->show(['id' => (string) $created['id']]);

        $this->assertSame(200, $status);
        $this->assertTrue($fetched['done']);
    }

    public function testDestroyRemovesTheTask(): void
    {
        [, $created] = $this->controller->store(['title' => 'Delete me']);

        [$deleteStatus] = $this->controller->destroy(['id' => (string) $created['id']]);
        [$showStatus] = $this->controller->show(['id' => (string) $created['id']]);

        $this->assertSame(200, $deleteStatus);
        $this->assertSame(404, $showStatus);
    }
}
```

```bash
vendor/bin/phpunit tests --testdox
```

```text
PHPUnit 11.5.56 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.5.9

......                                                              6 / 6 (100%)

Time: 00:00.010, Memory: 8.00 MB

Task Api (TaskApi\Tests\TaskApi)
 ✔ Creating a task returns 201 with the stored task
 ✔ Creating a task with blank title returns 422
 ✔ Index lists all created tasks
 ✔ Show returns 404 for missing task
 ✔ Complete marks task done
 ✔ Destroy removes the task

OK (6 tests, 13 assertions)
```

## What's missing for real production use

This project deliberately skips a few things a production API needs, each
covered later in the path: real HTTP routing wired to `$_SERVER` and a
front controller (Level 4's framework patterns), authentication on the
endpoints ([Security at Scale](../level-4/02-security-at-scale.md)),
pagination on `index()` once the table has more than a page of rows, and a
real database server instead of SQLite once concurrent writes matter.
Nothing about the *shape* of the code changes for any of that — the
repository/controller split is exactly what makes swapping SQLite for
MySQL, or adding auth middleware in front of the controller, additive
rather than a rewrite.

## Stretch goals

- Add a `PATCH /tasks/{id}` endpoint (`TaskController::update()`) that
  accepts a new `title` and updates the row, returning 404 if the id
  doesn't exist and 422 if the new title is blank. Add tests for both
  the happy path and both error cases.
- Wire the `Router` from this module into `demo.php` for real, mapping all
  five endpoints (`GET /tasks`, `GET /tasks/{id}`, `POST /tasks`,
  `PATCH /tasks/{id}/complete`, `DELETE /tasks/{id}`) and dispatching a
  handful of requests through `$router->dispatch()` instead of calling the
  controller directly.
- Add a `TaskRepository::search(string $query): array` method using
  `WHERE title LIKE :q` (bind `"%$query%"` as the parameter — never
  concatenate `$query` into the SQL string), exposed as
  `GET /tasks?q=...`, with a test proving a search for a substring that
  matches zero tasks returns an empty array rather than an error.
- Persist to a file-backed SQLite database (`Database::connect(__DIR__ .
  '/tasks.sqlite')`) instead of `:memory:` in `demo.php`, run it twice in a
  row, and observe that the second run's `GET /tasks` includes tasks
  created by the first run — then explain in a comment why the test suite
  deliberately keeps using `:memory:` instead.
