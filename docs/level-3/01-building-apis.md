# 01 · Building APIs

Level 2's [JSON/APIs module](../level-2/06-json-apis.md) covered encoding
and decoding JSON payloads one script at a time. A real API needs more than
that: it needs to look at an incoming URL and HTTP method and decide *which*
code should handle it — that's **routing**. Frameworks like Slim or
Laravel provide a router for you, but the underlying idea is simple enough to
build from scratch in a few dozen lines, and understanding it makes every
framework's routing layer far less mysterious.

## The front controller pattern

Instead of one PHP file per URL (`books.php`, `book-detail.php`, ...), a
modern API routes every request through a single entry point that inspects
the path and method, then dispatches to the right handler. This single
entry point is called a **front controller**.

```php
<?php
declare(strict_types=1);

final class Router
{
    /** @var array<string, array<string, callable>> */
    private array $routes = [];

    public function add(string $method, string $pattern, callable $handler): void
    {
        $this->routes[strtoupper($method)][$pattern] = $handler;
    }

    public function get(string $pattern, callable $handler): void
    {
        $this->add("GET", $pattern, $handler);
    }

    public function post(string $pattern, callable $handler): void
    {
        $this->add("POST", $pattern, $handler);
    }

    public function dispatch(string $method, string $uri): void
    {
        $path = parse_url($uri, PHP_URL_PATH) ?: "/";

        foreach ($this->routes[strtoupper($method)] ?? [] as $pattern => $handler) {
            $regex = $this->toRegex($pattern);
            if (preg_match($regex, $path, $matches)) {
                // Keep only the NAMED capture groups (route params), not the
                // numeric ones preg_match also returns.
                $params = array_filter($matches, fn($k) => is_string($k), ARRAY_FILTER_USE_KEY);
                $this->respond($handler(...$params));
                return;
            }
        }

        $this->respond(["error" => "Not found"], 404);
    }

    // Turns "/books/{id}" into a regex with a named capture group for "id",
    // so a URL segment can be pulled out and handed to the handler by name.
    private function toRegex(string $pattern): string
    {
        $escaped = preg_replace('#\{(\w+)\}#', '(?P<$1>[^/]+)', $pattern);
        return "#^" . $escaped . "$#";
    }

    private function respond(mixed $data, int $status = 200): void
    {
        if (!headers_sent()) {
            http_response_code($status);
            header("Content-Type: application/json");
        }
        echo json_encode($data, JSON_PRETTY_PRINT), "\n";
    }
}
```

## Registering routes and running it

```php
<?php
// index.php
require __DIR__ . "/Router.php";

$router = new Router();

$books = [
    1 => ["id" => 1, "title" => "The Pragmatic Programmer"],
    2 => ["id" => 2, "title" => "Clean Code"],
];

$router->get("/books", function () use ($books) {
    return array_values($books);
});

$router->get("/books/{id}", function (string $id) use ($books) {
    return $books[(int) $id] ?? ["error" => "Book not found"];
});

$router->post("/books", function () use (&$books) {
    // Unlike a form POST, a JSON API reads the raw request body itself --
    // $_POST is only populated for "application/x-www-form-urlencoded".
    $input = json_decode(file_get_contents("php://input"), true) ?? [];
    $id = count($books) + 1;
    $books[$id] = ["id" => $id, "title" => $input["title"] ?? "Untitled"];
    return $books[$id];
});

$router->dispatch($_SERVER["REQUEST_METHOD"], $_SERVER["REQUEST_URI"]);
```

Serve it with PHP's built-in development server — no Apache or nginx
needed to try this out:

```bash
php -S localhost:8000 -t . index.php
```

```bash
curl -s http://localhost:8000/books
```

```json
[
    {
        "id": 1,
        "title": "The Pragmatic Programmer"
    },
    {
        "id": 2,
        "title": "Clean Code"
    }
]
```

```bash
curl -s http://localhost:8000/books/1
```

```json
{
    "id": 1,
    "title": "The Pragmatic Programmer"
}
```

Requesting a path that matches no registered route (or a `GET` on a path
only registered for `POST`) falls through every `foreach` iteration and
hits the `404` fallback in `dispatch()` — there's no special-casing needed,
it's just what happens when nothing matches.

## A PHP-specific trap: route ordering

Because `dispatch()` checks patterns in the order they were registered,
overlapping patterns can shadow each other:

```php
<?php
// If registered in THIS order, "/books/featured" never matches its own
// route -- "/books/{id}" matches first, since "featured" satisfies
// "[^/]+" just as well as a numeric ID does.
$router->get("/books/{id}", fn(string $id) => ["id" => $id]);
$router->get("/books/featured", fn() => ["featured" => true]);

// Fix: register the more specific, literal route FIRST.
```

Real routers (Slim, Laravel, Symfony) solve this by sorting static segments
ahead of parameterized ones automatically; a hand-rolled router like this
one requires you to order routes carefully yourself — worth knowing before
reaching for a framework that hides the ordering problem.

## Content negotiation and status codes

An API should honor `Accept` headers and return meaningful HTTP status
codes — not just `200` for everything:

```php
<?php

function jsonResponse(mixed $data, int $status = 200): never
{
    http_response_code($status);
    header("Content-Type: application/json");
    echo json_encode($data);
    exit;
}

// 200 OK -- successful GET
jsonResponse(["id" => 1, "title" => "Clean Code"]);

// 201 Created -- successful POST that created a new resource
jsonResponse(["id" => 3, "title" => "Refactoring"], 201);

// 404 Not Found -- the resource doesn't exist
jsonResponse(["error" => "Book not found"], 404);

// 422 Unprocessable Entity -- the request was well-formed JSON but
// failed validation (e.g. missing required "title")
jsonResponse(["error" => "title is required"], 422);
```

## HTTP status code cheat sheet

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

## Why not just use a framework?

In production, most teams reach for [Slim](https://www.slimframework.com/)
or a full-stack framework rather than hand-rolling a router — they add
[middleware pipelines](07-middleware-patterns.md), request/response
objects, and route groups on top of the same core idea shown above:

```json
{
    "require": {
        "slim/slim": "^4.0",
        "slim/psr7": "^1.6"
    }
}
```

```php
<?php
// The framework version of the same route, for comparison --
// same core idea (method + pattern -> handler), less boilerplate.
$app->get("/books/{id}", function ($request, $response, $args) use ($books) {
    $response->getBody()->write(json_encode($books[(int) $args["id"]]));
    return $response->withHeader("Content-Type", "application/json");
});
```

Knowing how `dispatch()` works under the hood is what lets you read Slim's
source, debug a routing mismatch, or write a router for a constrained
environment where pulling in a framework isn't an option.

## Exercise

Extend the hand-rolled `Router` with a `PUT /books/{id}` route that reads a
JSON body and updates the matching book's title (return a `404` JSON body
if the ID doesn't exist), and a `DELETE /books/{id}` route that removes it
from the array and returns `["deleted" => true]`. Serve it with `php -S
localhost:8000` and exercise all four verbs with `curl -X GET`, `curl -X
POST`, `curl -X PUT`, and `curl -X DELETE`, printing each response.
