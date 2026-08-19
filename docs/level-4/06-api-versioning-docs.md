# 06 · API Versioning & Documentation

The [REST API project](../level-3/10-project-rest-api.md) returns whatever
shape `TaskRepository::hydrate()` produces. That's fine for one client you
control. The moment other teams or external clients depend on your API's
response shape, changing a field name breaks them the instant you deploy —
unless the change ships as a new, deliberately versioned response instead
of a silent mutation of the old one.

## Why versioning exists: a field rename is a breaking change

Renaming `is_done` to `done`, or adding a required field, looks like a
harmless refactor from inside your codebase. To every client parsing the
JSON response, it's a breaking change — code that reads `$response['is_done']`
gets `null` (or a fatal error, in a strictly-typed client) the instant the
old field disappears. Versioning lets both shapes exist simultaneously:
existing clients keep working against `v1`, new clients opt into `v2`.

## URL-based versioning with a transformer per version

The cleanest way to support multiple response shapes is to keep exactly one
internal domain object (`Task`) and use a small **transformer** per version
to decide how it's serialized — the domain model itself never has version
branches baked into it.

```php
<?php
// Task.php + transformers
declare(strict_types=1);

final class Task
{
    public function __construct(
        public int $id,
        public string $title,
        public bool $done,
        public string $createdAt,
    ) {}
}

interface TaskTransformer
{
    public function transform(Task $t): array;
}

final class TaskTransformerV1 implements TaskTransformer
{
    public function transform(Task $t): array
    {
        // v1 shipped with an integer flag and no timestamp -- frozen forever.
        return ['id' => $t->id, 'title' => $t->title, 'is_done' => $t->done ? 1 : 0];
    }
}

final class TaskTransformerV2 implements TaskTransformer
{
    public function transform(Task $t): array
    {
        // v2: boolean instead of 0/1, plus created_at -- the CURRENT shape.
        return ['id' => $t->id, 'title' => $t->title, 'done' => $t->done, 'created_at' => $t->createdAt];
    }
}
```

```php
<?php
// demo.php
declare(strict_types=1);

final class JsonResponse
{
    public function __construct(public int $status, public array $body) {}
}

function respondWithVersion(Task $task, string $version): JsonResponse
{
    $transformer = match ($version) {
        'v1' => new TaskTransformerV1(),
        'v2' => new TaskTransformerV2(),
        default => throw new InvalidArgumentException("Unknown API version: $version"),
    };
    return new JsonResponse(200, $transformer->transform($task));
}

$task = new Task(1, 'Write docs', true, '2026-08-18T10:00:00+00:00');

$v1 = respondWithVersion($task, 'v1');
echo "v1: " . json_encode($v1->body) . "\n";

$v2 = respondWithVersion($task, 'v2');
echo "v2: " . json_encode($v2->body) . "\n";
```

```text
v1: {"id":1,"title":"Write docs","is_done":1}
v2: {"id":1,"title":"Write docs","done":true,"created_at":"2026-08-18T10:00:00+00:00"}
```

In a real routed application, `$version` comes from the URL path
(`/v1/tasks/1` vs `/v2/tasks/1`, the most discoverable and cacheable
option) or an `Accept` header (`Accept: application/vnd.acme.v2+json`, more
"correct" per HTTP content negotiation but harder for clients to get right
and harder to test with a plain browser URL). Path-based versioning is the
pragmatic default for most APIs; header-based versioning shows up more in
API-first companies with strict HTTP discipline.

## Deprecation, not deletion

A version isn't retired the moment a new one ships — clients need time to
migrate. The practical pattern is: ship `v2`, keep `v1` working unchanged,
add a deprecation signal, and only remove `v1` after an announced date.

```php
<?php
declare(strict_types=1);

final class DeprecationHeader
{
    public static function forVersion(string $version): array
    {
        return match ($version) {
            'v1' => [
                'Deprecation' => 'true',
                'Sunset' => 'Wed, 31 Dec 2026 23:59:59 GMT',
                'Link' => '</docs/migrating-v1-to-v2>; rel="deprecation"',
            ],
            default => [],
        };
    }
}

$headers = DeprecationHeader::forVersion('v1');
foreach ($headers as $name => $value) {
    echo "$name: $value\n";
}
```

```text
Deprecation: true
Sunset: Wed, 31 Dec 2026 23:59:59 GMT
Link: </docs/migrating-v1-to-v2>; rel="deprecation"
```

`Deprecation` and `Sunset` are real, standardized HTTP response headers
(RFC 8594 / the Deprecation HTTP header draft) — a well-behaved client
library can detect them automatically and log a warning, giving consumers
advance notice through their own tooling rather than requiring them to read
a changelog.

## Documenting the API: OpenAPI as the source of truth

Hand-written prose documentation drifts from the actual API the moment
someone changes a field and forgets to update the docs page. **OpenAPI**
(formerly Swagger) describes an API as structured YAML/JSON that both
humans and tools can read — tools can generate interactive docs (Swagger UI,
Redoc), client SDKs, and even test-suite request validation from the same
file.

```yaml
# openapi.yaml (excerpt, describing the v2 task endpoints)
openapi: 3.0.3
info:
  title: Task API
  version: "2.0"
paths:
  /v2/tasks/{id}:
    get:
      summary: Fetch a single task
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        "200":
          description: The task
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: integer }
                  title: { type: string }
                  done: { type: boolean }
                  created_at: { type: string, format: date-time }
        "404":
          description: Task not found
```

Keeping this file in the same repository as the API code (and reviewing it
in the same pull request as an endpoint change) is what keeps documentation
from drifting — it's a diffable, testable artifact, not a separate wiki
page someone has to remember to update.

## PHP traps

**A transformer that silently falls through to a default version hides
client bugs.** `respondWithVersion()` above throws on an unknown version
rather than quietly defaulting to `v2` — a client that mistypes
`/v3/tasks/1` should get a clear 4xx error, not a response shape it never
asked for and may not know how to parse.

**Sharing one transformer instance across requests can leak state if it
isn't stateless.** The transformers above hold no properties and are safe
to reuse, but the moment a transformer accumulates any per-request state
(a counter, a cache of resolved data), reusing the same instance across
concurrent requests in a long-running worker (not a fresh PHP-per-request
model) becomes a data-leak risk between unrelated requests.

**Versioning the URL but not the OpenAPI spec's `info.version`** leaves
generated documentation and client SDKs pointing at the wrong contract —
treat the spec file itself as versioned alongside the code, not as a
one-time document.

## Versioning & docs cheat sheet

| Concept | Purpose |
|---|---|
| URL versioning (`/v1/...`, `/v2/...`) | Simple, cacheable, discoverable in a browser |
| Header versioning (`Accept: ...+json`) | "Correct" per HTTP content negotiation, harder to test casually |
| Transformer per version | One domain model, multiple serialized shapes |
| `Deprecation` / `Sunset` headers | Machine-readable notice before removing an old version |
| OpenAPI spec | Structured, diffable source of truth; generates docs + SDKs |
| Throwing on unknown version | Surfaces client mistakes instead of guessing |

## Exercise

Add a `TaskTransformerV3` that includes a nested `meta` object
(`['meta' => ['api_version' => 'v3', 'generated_at' => date(DATE_ATOM)]]`
alongside the existing `id`/`title`/`done`/`created_at` fields), wire it
into `respondWithVersion()`, and write a small script that requests all
three versions for the same `Task` and prints each response body — confirm
`v1`'s `is_done` is an integer (`1`/`0`, not `true`/`false`) while `v2` and
`v3` use real booleans, and that only `v3`'s output contains a `meta` key.
