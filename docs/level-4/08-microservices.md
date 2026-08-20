# 08 · Working with Microservices in PHP

Every project in this path so far has been a **monolith** — one codebase,
one database, one deploy. A **microservices** architecture splits an
application into independently deployable services, each owning its own
data, communicating over the network (usually HTTP or a message queue).
PHP is a perfectly reasonable choice for individual services in such a
system. This module covers the pattern that matters most once services
depend on each other over a network: what happens when one of them is
slow or down.

## Modeling service boundaries

Two services in the same process (simulated here without real HTTP servers,
so the example stays runnable) still illustrate the essential shape: each
owns its own data, and one calls the other only through a defined
interface — never by reaching into the other's database directly.

```php
<?php
// UserService.php
declare(strict_types=1);

interface UserService
{
    public function find(int $id): ?array;
}

final class RealUserService implements UserService
{
    private PDO $pdo;

    public function __construct()
    {
        $this->pdo = new PDO('sqlite::memory:');
        $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $this->pdo->exec('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');
        $this->pdo->exec("INSERT INTO users VALUES (1, 'Ada'), (2, 'Grace')");
    }

    public function find(int $id): ?array
    {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = ?');
        $stmt->execute([$id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        return $row === false ? null : $row;
    }
}
```

In a real deployment, `RealUserService::find()` would be an HTTP call
(`GET http://user-service/users/1`) or a message over a queue — the
*interface* is what lets `OrderService` (below) depend on "something that
can find a user by id" without caring whether that's an in-process call, an
HTTP request, or eventually a completely different implementation.

## The problem network calls introduce: cascading failure

A monolith calling its own function either works or throws immediately. A
service calling *another service over the network* can also **hang** —
and if every request to a failing downstream service waits out a long
timeout before giving up, a struggling service can take down everything
that calls it, one slow request at a time, by exhausting the caller's own
worker pool.

```php
<?php
// OrderService.php
declare(strict_types=1);

final class OrderService
{
    private PDO $pdo;

    public function __construct(private UserService $users, private CircuitBreaker $breaker)
    {
        $this->pdo = new PDO('sqlite::memory:');
        $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $this->pdo->exec('CREATE TABLE orders (id INTEGER PRIMARY KEY, user_id INTEGER, total INTEGER)');
        $this->pdo->exec("INSERT INTO orders VALUES (1, 1, 2500)");
    }

    public function findWithUser(int $orderId): array
    {
        $stmt = $this->pdo->prepare('SELECT * FROM orders WHERE id = ?');
        $stmt->execute([$orderId]);
        $order = $stmt->fetch(PDO::FETCH_ASSOC);
        if ($order === false) {
            throw new RuntimeException("Order not found");
        }

        // Calling the OTHER service through the circuit breaker, not directly.
        $user = $this->breaker->call(fn() => $this->users->find((int) $order['user_id']));
        return ['order' => $order, 'user' => $user];
    }
}
```

## The circuit breaker: failing fast instead of failing slow

A **circuit breaker** tracks recent failures to a downstream service and,
past a threshold, stops even *trying* to call it for a while — failing
immediately instead of waiting out a timeout on every request. This
protects the caller (no more piling-up slow requests) and gives the
downstream service breathing room to recover instead of being hit with
retries from every caller simultaneously.

```php
<?php
// CircuitBreaker.php
declare(strict_types=1);

final class ServiceUnavailableException extends RuntimeException {}

final class CircuitBreaker
{
    private int $failures = 0;
    private bool $open = false;

    public function __construct(private int $threshold = 3) {}

    public function call(callable $fn): mixed
    {
        if ($this->open) {
            throw new ServiceUnavailableException("Circuit open -- failing fast");
        }
        try {
            $result = $fn();
            $this->failures = 0;   // a success resets the failure count
            return $result;
        } catch (Throwable $e) {
            $this->failures++;
            if ($this->failures >= $this->threshold) {
                $this->open = true;
                echo "  [circuit breaker] tripped open after {$this->failures} failures\n";
            }
            throw $e;
        }
    }
}
```

## Seeing it trip

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/UserService.php';
require __DIR__ . '/CircuitBreaker.php';
require __DIR__ . '/OrderService.php';

$orders = new OrderService(new RealUserService(), new CircuitBreaker(threshold: 2));
$result = $orders->findWithUser(1);
echo "Order #{$result['order']['id']} for {$result['user']['name']}, total {$result['order']['total']}\n";

echo "\n-- Simulating the user service going down --\n";

final class FlakyUserService implements UserService
{
    public function find(int $id): ?array { throw new RuntimeException('connection refused'); }
}

$flakyOrders = new OrderService(new FlakyUserService(), new CircuitBreaker(threshold: 2));

for ($i = 1; $i <= 3; $i++) {
    try {
        $flakyOrders->findWithUser(1);
    } catch (ServiceUnavailableException $e) {
        echo "Attempt $i: circuit open, failed fast: {$e->getMessage()}\n";
    } catch (RuntimeException $e) {
        echo "Attempt $i: real failure: {$e->getMessage()}\n";
    }
}
```

```text
Order #1 for Ada, total 2500

-- Simulating the user service going down --
Attempt 1: real failure: connection refused
  [circuit breaker] tripped open after 2 failures
Attempt 2: real failure: connection refused
Attempt 3: circuit open, failed fast: Circuit open -- failing fast
```

The first two failures are the *real* `RuntimeException` from
`FlakyUserService` — the breaker lets them through while counting. The
third call never reaches `FlakyUserService::find()` at all: the breaker is
open, so it throws `ServiceUnavailableException` immediately. In a real
network call, that's the difference between failing in microseconds versus
waiting out a multi-second timeout on every single request while the
downstream service is struggling.

## PHP traps

**An open circuit breaker with no recovery path stays open forever.** This
minimal implementation never closes back — a production circuit breaker
adds a **half-open** state: after a cooldown period, let exactly one
request through as a probe; if it succeeds, close the circuit (resume
normal calls); if it fails, stay open and wait again. Without that,
`$open = true` here is permanent for the object's lifetime.

**Wrapping *every* downstream call in the same shared breaker conflates
unrelated failures.** If `OrderService` also called a `PaymentService`
through the *same* `CircuitBreaker` instance, three failed payment calls
would trip the breaker and start failing fast on unrelated user lookups
too. Each downstream dependency needs its own breaker instance.

**A network timeout is not the same failure mode as a clean exception**,
and code that only tests the latter (as this module's demo does, for
runnability) can be surprised in production — a hung connection needs an
explicit timeout configured on the HTTP client itself (e.g. `CURLOPT_TIMEOUT`
or Guzzle's `timeout` option) so the circuit breaker's `try`/`catch` is
even reachable; without a timeout, the call can hang indefinitely and never
throw at all.

## Microservices cheat sheet

| Concept | Purpose |
|---|---|
| Service interface (`UserService`) | Caller depends on a contract, not a transport (HTTP vs. in-process) |
| Each service owns its data | No service reaches into another's database directly |
| Circuit breaker | Fails fast after repeated failures instead of piling up slow requests |
| Closed / open / half-open | Normal / failing-fast / probing-for-recovery states |
| Per-dependency breaker | One breaker per downstream service, not shared globally |
| Client-side timeout | What actually makes a hung call throw so the breaker can react |

## Exercise

Add a half-open state to `CircuitBreaker`: a `cooldownSeconds` constructor
parameter and an `openedAt` timestamp set when the circuit trips. Change
`call()` so that once `time() - $openedAt >= $cooldownSeconds`, the next
call is let through as a probe (not immediately rejected) — if it succeeds,
close the circuit (`$open = false`, reset failures); if it fails, reset
`$openedAt` and stay open. Test it with a fake clock (pass `$now` into
`call()` instead of using `time()` directly, for a deterministic test): trip
the breaker, confirm calls fail fast before the cooldown, then confirm a
call after the cooldown is attempted again.
