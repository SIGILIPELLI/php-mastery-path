# 07 · Middleware Patterns

Every framework request cycle needs cross-cutting concerns handled before
and after the actual route logic runs: authentication, logging, rate
limiting, CORS headers. Sprinkling those checks into every controller is
repetitive and easy to forget. **Middleware** solves this by wrapping the
request in layers — each layer can inspect or modify the request, decide
whether to continue, and inspect or modify the response on the way back out.

## The shape of a middleware pipeline

Each middleware receives the request and a `$next` callable representing
"the rest of the pipeline." It decides whether to call `$next($request)` at
all, and what to do with the response that comes back.

```php
<?php
// Request.php, Response.php
declare(strict_types=1);

final class Request
{
    public array $attributes = [];
    public function __construct(public string $method, public string $path, public array $headers = []) {}
}

final class Response
{
    public function __construct(public int $status, public string $body) {}
}
```

```php
<?php
// Middleware.php
declare(strict_types=1);

interface Middleware
{
    public function handle(Request $request, callable $next): Response;
}
```

## Two concrete middleware: logging and auth

```php
<?php
// LoggingMiddleware.php
declare(strict_types=1);

final class LoggingMiddleware implements Middleware
{
    public function handle(Request $request, callable $next): Response
    {
        $start = microtime(true);
        $response = $next($request);              // let the rest of the pipeline run
        $ms = round((microtime(true) - $start) * 1000, 2);
        echo "[log] {$request->method} {$request->path} -> {$response->status} ({$ms}ms)\n";
        return $response;
    }
}
```

```php
<?php
// AuthMiddleware.php
declare(strict_types=1);

final class AuthMiddleware implements Middleware
{
    public function handle(Request $request, callable $next): Response
    {
        $token = $request->headers['Authorization'] ?? null;

        if ($token !== 'Bearer secret-token') {
            echo "[auth] rejected request to {$request->path}\n";
            return new Response(401, 'Unauthorized');   // short-circuits -- $next is never called
        }

        $request->attributes['user'] = 'ada';
        echo "[auth] accepted, user=ada\n";
        return $next($request);
    }
}
```

`AuthMiddleware` demonstrates the key power of the pattern: it can refuse to
call `$next()` at all, stopping the pipeline dead and returning its own
response — the controller never runs for a rejected request.

## Building the pipeline

```php
<?php
// Pipeline.php
declare(strict_types=1);

final class Pipeline
{
    /** @var Middleware[] */
    private array $middleware = [];

    public function pipe(Middleware $m): static
    {
        $this->middleware[] = $m;
        return $this;
    }

    public function handle(Request $request, callable $finalHandler): Response
    {
        // Build the chain from the INSIDE out: start with the controller,
        // then wrap each middleware around what came before, in reverse
        // registration order, so the FIRST piped middleware runs FIRST.
        $chain = array_reduce(
            array_reverse($this->middleware),
            fn(callable $next, Middleware $m) => fn(Request $r) => $m->handle($r, $next),
            $finalHandler
        );
        return $chain($request);
    }
}
```

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/Request.php';
require __DIR__ . '/Response.php';
require __DIR__ . '/Middleware.php';
require __DIR__ . '/LoggingMiddleware.php';
require __DIR__ . '/AuthMiddleware.php';
require __DIR__ . '/Pipeline.php';

$controller = fn(Request $r) => new Response(200, "Hello, {$r->attributes['user']}!");

$pipeline = (new Pipeline())->pipe(new LoggingMiddleware())->pipe(new AuthMiddleware());

echo "-- Request 1: valid token --\n";
$req1 = new Request('GET', '/profile', ['Authorization' => 'Bearer secret-token']);
$res1 = $pipeline->handle($req1, $controller);
echo "Response: {$res1->status} {$res1->body}\n\n";

echo "-- Request 2: missing token --\n";
$req2 = new Request('GET', '/profile');
$res2 = $pipeline->handle($req2, $controller);
echo "Response: {$res2->status} {$res2->body}\n";
```

```text
-- Request 1: valid token --
[auth] accepted, user=ada
[log] GET /profile -> 200 (0.31ms)
Response: 200 Hello, ada!

-- Request 2: missing token --
[auth] rejected request to /profile
[log] GET /profile -> 401 (0.09ms)
Response: 401 Unauthorized
```

Watch the ordering: `LoggingMiddleware` was piped *first*, so it's the
outermost layer — its "before" code (`$start = microtime(true)`) runs
first, but its "after" code (the `echo "[log]..."` line) runs *last*,
after everything inside it — including a short-circuited `AuthMiddleware`
rejection — has finished. That's what makes it able to log the *real*
status code and timing no matter which layer produced the response.

## PHP traps

**Forgetting to return `$next($request)`'s result.** A middleware that
calls `$next($request);` as a statement instead of `return $next($request);`
discards the response and returns `null` from `handle()` — the caller then
crashes trying to read `->status` off `null`. Every middleware that
continues the chain must return what `$next()` gave it (optionally after
modifying it).

**Mutating the request object works only because it's shared by reference.**
`$request->attributes['user'] = 'ada'` in `AuthMiddleware` is visible to
every layer *inside* it (including the controller) because `Request` is a
plain object, passed by handle, not by value. If `Request` were an
immutable value object instead (a legitimate, arguably safer design), each
middleware would need to pass a *new* `$request` into `$next()` rather than
mutating in place — check which style a framework uses before assuming
attributes "just appear."

**Registration order is easy to get backwards.** Piping `AuthMiddleware`
*before* `LoggingMiddleware` (swap the two `->pipe()` calls above) makes
auth the outermost layer — a rejected request would never reach the logger
at all, silently losing visibility into failed auth attempts. Order
cross-cutting concerns deliberately: logging outermost (see everything),
then auth, then anything that assumes an authenticated user.

## Middleware cheat sheet

| Concept | What it means here |
|---|---|
| `Middleware::handle($request, $next)` | One layer; decides whether/how to call `$next` |
| `$next($request)` | Continue to the next layer (or the controller) |
| Not calling `$next()` | Short-circuit — return a response immediately (auth failure, rate limit) |
| `Pipeline::pipe()` order | First piped = outermost = runs first going in, last coming out |
| `array_reduce` + `array_reverse` | Standard trick for building a nested chain from a flat list |
| Mutating `$request->attributes` | How inner layers see data set by outer layers (e.g. the authenticated user) |

## Exercise

Write a `RateLimitMiddleware` that tracks request counts per
`$request->attributes['user'] ?? 'anonymous'` in an in-memory array keyed by
user, and returns `new Response(429, 'Too Many Requests')` (without calling
`$next()`) once a user exceeds 3 requests within the object's lifetime. Pipe
it *after* `AuthMiddleware` (so it runs with the user already attached) and
*before* the controller. Send 5 requests with a valid token through the
pipeline in a loop and print each response's status — confirm the first 3
succeed and the last 2 are rate-limited.
