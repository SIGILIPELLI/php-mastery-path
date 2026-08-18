# 05 · Performance & Caching

Most PHP performance problems aren't about the language being slow — they're
about doing the same expensive work over and over. A database query that
runs on every request, a report recomputed from scratch each time it's
viewed, an API call repeated for data that changes once a day. Caching fixes
that class of problem; OPcache fixes a different one (PHP recompiling your
source files on every request). This module covers both.

## OPcache: caching compiled bytecode, not your data

Every time a plain PHP process runs a script, it parses the source and
compiles it to bytecode before executing it — work repeated on every single
request, for source that hasn't changed. **OPcache** (bundled with PHP,
usually on by default under PHP-FPM) caches that compiled bytecode in shared
memory so subsequent requests skip straight to execution.

```php
<?php
declare(strict_types=1);

var_dump(function_exists('opcache_get_status'));
```

```text
bool(true)
```

`opcache_get_status()` returns cache hit/miss counts and memory usage when
OPcache is active — useful for confirming it's actually doing something in
production. The CLI SAT (`php script.php`) typically runs with
`opcache.enable_cli=Off`, since a one-shot script gains nothing from caching
bytecode it will only execute once; the setting that matters is
`opcache.enable` under PHP-FPM/Apache, where the same file is compiled once
and reused across thousands of requests. There's nothing to configure in
application code — OPcache is a `php.ini` concern (`opcache.enable`,
`opcache.memory_consumption`, `opcache.validate_timestamps`), not an API you
call.

## Application-level caching: a `remember()` pattern

OPcache speeds up *running your code*. It does nothing for the cost of
*what your code does* — a slow query, an external API call, a heavy
computation. For that you cache the **result**, keyed by something that
identifies the request, with a time-to-live (TTL) after which it's
considered stale.

```php
<?php
// FileCache.php
declare(strict_types=1);

final class FileCache
{
    public function __construct(private string $dir)
    {
        if (!is_dir($this->dir)) {
            mkdir($this->dir, 0777, true);
        }
    }

    private function path(string $key): string
    {
        return $this->dir . '/' . hash('sha256', $key) . '.cache';
    }

    /** Return the cached value for $key, computing and storing it if missing or expired. */
    public function remember(string $key, int $ttlSeconds, callable $compute): mixed
    {
        $file = $this->path($key);

        if (is_file($file)) {
            $payload = unserialize(file_get_contents($file));
            if ($payload['expires'] > time()) {
                return $payload['value'];
            }
        }

        $value = $compute();
        file_put_contents($file, serialize(['value' => $value, 'expires' => time() + $ttlSeconds]));
        return $value;
    }
}
```

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/FileCache.php';

$cache = new FileCache(sys_get_temp_dir() . '/php_cache_demo');

function expensiveSum(int $n): int
{
    usleep(200_000); // simulate a slow query or API call
    return array_sum(range(1, $n));
}

$start = microtime(true);
$result1 = $cache->remember('sum_1000', 5, fn() => expensiveSum(1000));
$elapsed1 = microtime(true) - $start;

$start = microtime(true);
$result2 = $cache->remember('sum_1000', 5, fn() => expensiveSum(1000)); // cache hit
$elapsed2 = microtime(true) - $start;

printf("First call:  result=%d elapsed=%.3fs\n", $result1, $elapsed1);
printf("Second call: result=%d elapsed=%.3fs\n", $result2, $elapsed2);
```

```text
First call:  result=500500 elapsed=0.207s
Second call: result=500500 elapsed=0.000s
```

The second call skips `expensiveSum()` entirely — it's a cache hit that
costs a file read and an `unserialize()` instead of 200ms of "work." This
`FileCache` is deliberately simple so it's runnable with no extra
infrastructure. In production, reach for a real cache backend instead:

- **APCu** — an in-memory key/value cache local to a single server process
  (`apcu_fetch()`/`apcu_store()`). Fast, but not shared across multiple
  app servers, and wiped on deploy/restart.
- **Redis / Memcached** — networked, shared across every app server, and
  the standard choice once you're running more than one PHP instance.
  Libraries like `symfony/cache` and `predis/predis` wrap these behind the
  same kind of `get`/`set`/`remember` interface shown above, so the calling
  code barely changes when you swap the backend.

## Cache invalidation: the actually-hard part

"There are only two hard things in computer science: cache invalidation and
naming things." Storing a value is easy. Knowing *when it's wrong* is not.

```php
<?php
declare(strict_types=1);
require __DIR__ . '/FileCache.php';

$dir = sys_get_temp_dir() . '/php_cache_demo2';
$cache = new FileCache($dir);

// Scenario: caching a user's profile, keyed by user id.
function loadProfile(int $userId): array
{
    return ['id' => $userId, 'name' => 'Ada Lovelace', 'loaded_at' => date('H:i:s')];
}

$userId = 42;
$profile = $cache->remember("profile:$userId", 60, fn() => loadProfile($userId));
echo "Cached profile: {$profile['name']} (loaded_at {$profile['loaded_at']})\n";

// The user just updated their name. The cache doesn't know that -- it will
// keep serving the stale value for up to 60 more seconds unless something
// explicitly invalidates the key.
function invalidateProfile(string $dir, int $userId): void
{
    $key = "profile:$userId";
    $file = $dir . '/' . hash('sha256', $key) . '.cache';
    if (is_file($file)) {
        unlink($file);
    }
}
invalidateProfile($dir, $userId);
echo "Cache entry explicitly invalidated after the profile update.\n";
```

```text
Cached profile: Ada Lovelace (loaded_at 15:28:05)
Cache entry explicitly invalidated after the profile update.
```

Two strategies handle this in practice: **short TTLs** (accept a small
staleness window in exchange for never writing invalidation code), or
**explicit invalidation** (delete/overwrite the key the moment the
underlying data changes, as above). Most real systems mix both — a
generous TTL as a safety net, plus explicit invalidation on writes for data
where staleness is visibly wrong (a price, an account balance).

## PHP traps

**`unserialize()` on untrusted input is a security hole**, not just a
caching detail — PHP's serialized format can trigger `__wakeup()`/`__destruct()`
on classes present in your codebase (a POP-chain attack). The cache above is
safe because *your own process* wrote the file. Never `unserialize()` a
value that came from a cache shared with, or writable by, anything you don't
fully trust — prefer `json_encode()`/`json_decode()` for cached data that
doesn't need to preserve PHP object identity.

**A cache stampede** happens when a popular key expires and many concurrent
requests all see a miss at once, and all recompute the expensive value
simultaneously — multiplying load exactly when the system is already
regenerating something costly. Production caches usually guard against this
with a lock ("only one request recomputes; others wait or serve the stale
value") or by staggering TTLs with jitter (`$ttl + random_int(0, 30)`) so
keys don't all expire at the same instant.

**Caching a mutable object by reference defeats the point.** If `$compute()`
returns an object and the caller mutates it, the *next* `remember()` call
(if it happens to still hold that same in-process reference, e.g. with an
array-based cache instead of serialize/unserialize) can hand back the
mutated version instead of the original. Serializing to a file, as above,
sidesteps this because `unserialize()` always produces a fresh object graph
— one more reason file/Redis-backed caches are safer defaults than a bare
in-process array.

## Caching cheat sheet

| Tool | Caches | Scope | Survives restart? |
|------|--------|-------|--------------------|
| OPcache | Compiled bytecode | Per server | No (rebuilt on first request) |
| APCu | Arbitrary values | Per server process | No |
| Redis / Memcached | Arbitrary values | Shared across servers | Yes (Redis, if persistence configured) |
| File cache (this module) | Arbitrary values | Per filesystem | Yes |
| `remember($key, $ttl, $fn)` | Pattern: check cache, else compute + store | — | — |

## Exercise

Extend `FileCache` with a `forget(string $key): void` method that deletes a
key's cache file if it exists (reuse the `path()` logic). Then write a small
script that: caches the result of a `slowIsPrime(int $n): bool` function
(add an `usleep(150_000)` to simulate work) for `n = 97`; times the first
call and confirms it's slow; times a second call and confirms it's fast;
calls `forget()`; times a third call and confirms it's slow again because
the entry was invalidated. Print all three elapsed times.
