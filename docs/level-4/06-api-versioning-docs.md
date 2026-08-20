# 06 · API Versioning & Documentation

An API with real clients can't just change shape whenever the backend needs
to — a mobile app on last year's release still expects last year's response
format. **Versioning** lets old and new response shapes coexist, and
**generated documentation** keeps the description of an API from silently
drifting away from what the code actually does.

## Content-negotiated versioning

There's more than one way to version an API (`/v1/users` in the URL,
`?version=2` in the query string), but header-based negotiation via the
`Accept` header keeps URLs stable while still letting a client pin an exact
version:

```php
<?php
declare(strict_types=1);

interface ApiVersion {
    public function transform(array $data): array;
}

final class V1Response implements ApiVersion {
    public function transform(array $data): array {
        return ['id' => $data['id'], 'name' => $data['full_name']];
    }
}

final class V2Response implements ApiVersion {
    public function transform(array $data): array {
        return [
            'id' => $data['id'],
            'name' => ['first' => explode(' ', $data['full_name'])[0], 'full' => $data['full_name']],
            'meta' => ['version' => 2],
        ];
    }
}
```

Each version's `transform()` maps the *same* internal data (`$data`) to a
*different* public shape. Nothing about how the user is stored or fetched
changes between versions — only how it's presented, which is exactly the
separation that lets v1 clients keep working while v2 is developed.

```php
<?php
final class Router {
    private array $versions = [];
    public function register(string $version, ApiVersion $handler): void {
        $this->versions[$version] = $handler;
    }
    public function handle(string $acceptHeader, array $data): array {
        // Accept: application/vnd.myapi.v2+json
        if (!preg_match('/vnd\.myapi\.(v\d+)\+json/', $acceptHeader, $m)) {
            throw new InvalidArgumentException("Unrecognized Accept header: $acceptHeader");
        }
        $version = $m[1];
        if (!isset($this->versions[$version])) {
            throw new RuntimeException("Unsupported API version: $version");
        }
        return $this->versions[$version]->transform($data);
    }
}

$router = new Router();
$router->register('v1', new V1Response());
$router->register('v2', new V2Response());

$user = ['id' => 42, 'full_name' => 'Ada Lovelace'];

echo json_encode($router->handle('application/vnd.myapi.v1+json', $user), JSON_PRETTY_PRINT) . "\n";
echo json_encode($router->handle('application/vnd.myapi.v2+json', $user), JSON_PRETTY_PRINT) . "\n";

try {
    $router->handle('application/vnd.myapi.v9+json', $user);
} catch (RuntimeException $e) {
    echo "Caught: " . $e->getMessage() . "\n";
}
```

```text
{
    "id": 42,
    "name": "Ada Lovelace"
}
{
    "id": 42,
    "name": {
        "first": "Ada",
        "full": "Ada Lovelace"
    },
    "meta": {
        "version": 2
    }
}
Caught: Unsupported API version: v9
```

A vendor-specific media type (`application/vnd.myapi.v2+json`, rather than a
raw `application/json`) makes version negotiation explicit and cacheable at
the HTTP layer — proxies and CDNs can key on it, unlike a custom header that
generic HTTP infrastructure doesn't know to look at.

## Generating docs from the code, not beside it

Hand-written API docs (a wiki page, a separate Markdown file) rot the moment
someone changes an endpoint and forgets to update the doc. PHP 8's
**attributes** let documentation live as metadata directly on the code that
implements it, so a doc generator can read it back via `Reflection` — the
same mechanism [Level 3's DI module](../level-3/08-dependency-injection.md)
used to inspect constructors.

```php
<?php
declare(strict_types=1);

#[Attribute]
final class RouteDoc {
    public function __construct(
        public string $method,
        public string $path,
        public string $summary,
    ) {}
}

final class UserController {
    #[RouteDoc('GET', '/users/{id}', 'Fetch a single user by id')]
    public function show(int $id): array { return ['id' => $id]; }

    #[RouteDoc('POST', '/users', 'Create a new user')]
    public function store(array $data): array { return $data; }
}
```

```php
<?php
function generateDocs(string $class): array {
    $reflection = new ReflectionClass($class);
    $docs = [];
    foreach ($reflection->getMethods() as $method) {
        foreach ($method->getAttributes(RouteDoc::class) as $attr) {
            $route = $attr->newInstance();
            $docs[] = [
                'method' => $route->method,
                'path' => $route->path,
                'summary' => $route->summary,
                'handler' => $method->getName(),
            ];
        }
    }
    return $docs;
}

foreach (generateDocs(UserController::class) as $entry) {
    printf("%-5s %-15s %-30s -> %s()\n", $entry['method'], $entry['path'], $entry['summary'], $entry['handler']);
}
```

```text
GET   /users/{id}     Fetch a single user by id      -> show()
POST  /users          Create a new user              -> store()
```

`$attr->newInstance()` actually constructs a real `RouteDoc` object from the
attribute's declared arguments — attributes aren't just string metadata,
they're instantiable classes, so the doc generator gets typed, validated
data rather than parsing a docblock comment with regex. This is the same
mechanism behind real tools like `zircote/swagger-php`, which walk attributes
like this one to emit a full OpenAPI JSON/YAML spec.

## PHP traps

**Attribute arguments are evaluated once, at `newInstance()` time, not at
class-load time.** Declaring `#[RouteDoc('GET', '/users/{id}', SOME_CONST)]`
works fine because PHP resolves constants when it builds the instance — but
attributes cannot run arbitrary expressions or call functions in their
argument list, only literals, constants, and (as of PHP 8.1) enum cases.
Trying `#[RouteDoc(strtoupper('get'), ...)]` is a compile error.

**A version transformer that mutates the original data breaks the *other*
version's transformer.** If `V1Response::transform()` did
`$data['name'] = $data['full_name']; unset($data['full_name']);` and `$data`
were passed by reference elsewhere, a second call for `V2Response` on the
same underlying data would find `full_name` already gone. Keeping
`transform()` pure — read `$data`, return a new array, never mutate the
input — is what makes registering multiple version handlers against the same
source data safe.

**Forgetting to update the `Accept` regex when adding v10+ silently breaks
matching.** `(v\d+)` handles `v1` through `v99` correctly, but a version
scheme that switches to letters or dates (`v2024-01`) needs the pattern
updated — an easy thing to miss since the code keeps "working" for existing
versions and only the new one 404s.

## Versioning & docs cheat sheet

| Concept | Mechanism | Example |
|---|---|---|
| Header-based versioning | `Accept: application/vnd.*.vN+json` | `V1Response` vs. `V2Response` |
| Version isolation | Each version has its own pure `transform()` | Same `$data`, different shape out |
| Unsupported version | Fail with a clear error, not a silent default | `RuntimeException` in `handle()` |
| Docs as metadata | PHP 8 attributes (`#[RouteDoc(...)]`) | Read via `Reflection` |
| Doc generation | `getAttributes()` + `newInstance()` | `generateDocs()` |
| Real-world equivalent | `zircote/swagger-php`, OpenAPI spec generation | Attributes -> JSON/YAML spec |

## Exercise

Add a `V3Response` that includes a `links` field
(`['self' => "/users/{$data['id']}"]`) alongside the `V2Response` shape, and
register it under `'v3'`. Then extend `RouteDoc` with an optional
`deprecated: bool = false` constructor parameter, mark `UserController::show()`
as `#[RouteDoc('GET', '/users/{id}', 'Fetch a single user by id',
deprecated: true)]`, and update `generateDocs()` to print `[DEPRECATED]`
before any entry where `deprecated` is `true`.
