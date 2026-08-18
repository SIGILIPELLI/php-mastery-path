# 08 · Dependency Injection Basics

A class that reaches inside itself to create its own collaborators
(`new MysqlLogger()` buried in a constructor) is welded to that concrete
implementation — you can't swap it for a test double, a different backend,
or a decorated version without editing the class. **Dependency injection**
flips this: a class *declares* what it needs (through its constructor,
typically as interfaces) and something else — the caller, or a **container**
— supplies the concrete instance. [Level 3's mocking module](04-testing-advanced-mocking.md)
already relied on this; here's why it works and how a container automates it.

## Constructor injection

```php
<?php
// interfaces + implementations
declare(strict_types=1);

interface Logger { public function log(string $msg): void; }
final class EchoLogger implements Logger {
    public function log(string $msg): void { echo "[LOG] $msg\n"; }
}

interface Clock { public function now(): DateTimeImmutable; }
final class SystemClock implements Clock {
    public function now(): DateTimeImmutable { return new DateTimeImmutable(); }
}
```

```php
<?php
// ReportGenerator.php
declare(strict_types=1);

final class ReportGenerator
{
    public function __construct(private Logger $logger, private Clock $clock) {}

    public function generate(): string
    {
        $this->logger->log('generating report');
        return 'Report as of ' . $this->clock->now()->format('Y-m-d');
    }
}
```

`ReportGenerator` never writes `new EchoLogger()` or `new SystemClock()`
anywhere — it only knows about the `Logger` and `Clock` *interfaces*. That's
what makes it possible to hand it a `FakeClock` in a test (fixing "now" to a
known date) without touching this class at all, exactly the pattern
[the mocking module](04-testing-advanced-mocking.md) exercised.

## Manual wiring: fine for a handful of classes

```php
<?php
declare(strict_types=1);

$logger = new EchoLogger();
$clock = new SystemClock();
$report = new ReportGenerator($logger, $clock);

echo $report->generate() . "\n";
```

```text
[LOG] generating report
Report as of 2026-08-18
```

This is dependency injection — nothing more exotic is *required*. A
container becomes worth the complexity only once wiring by hand turns
tedious: a class with five dependencies, each of which has its own
dependencies, repeated across dozens of places that need one.

## A minimal autowiring container

A **container** centralizes "how do I build a `Logger`?" in one place, and
can use PHP's `Reflection` API to inspect a class's constructor and build
its dependencies automatically — **autowiring**.

```php
<?php
// Container.php
declare(strict_types=1);

final class Container
{
    private array $bindings = [];
    private array $instances = [];

    /** Register how to build $abstract (an interface or class name). */
    public function bind(string $abstract, callable $factory): void
    {
        $this->bindings[$abstract] = $factory;
    }

    public function get(string $abstract): mixed
    {
        if (isset($this->instances[$abstract])) {
            return $this->instances[$abstract];    // reuse the same instance
        }
        if (isset($this->bindings[$abstract])) {
            return $this->instances[$abstract] = ($this->bindings[$abstract])($this);
        }
        return $this->instances[$abstract] = $this->autowire($abstract);
    }

    private function autowire(string $class): object
    {
        $reflection = new ReflectionClass($class);
        $constructor = $reflection->getConstructor();
        if ($constructor === null) {
            return new $class();
        }

        $args = [];
        foreach ($constructor->getParameters() as $param) {
            $type = $param->getType();
            if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
                $args[] = $this->get($type->getName());   // recurse into ITS dependencies
            } else {
                throw new RuntimeException("Cannot autowire scalar parameter \${$param->getName()}");
            }
        }
        return $reflection->newInstanceArgs($args);
    }
}
```

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/Container.php';

$container = new Container();
$container->bind(Logger::class, fn() => new EchoLogger());
$container->bind(Clock::class, fn() => new SystemClock());

$report = $container->get(ReportGenerator::class);   // autowired: constructor deps resolved via reflection
echo $report->generate() . "\n";

$report2 = $container->get(ReportGenerator::class);
var_dump($report === $report2);   // container cached the instance -> same object
```

```text
[LOG] generating report
Report as of 2026-08-18
bool(true)
```

`Container::get(ReportGenerator::class)` never appears in the `bindings`
array — the container falls through to `autowire()`, inspects
`ReportGenerator`'s constructor via reflection, sees it needs a `Logger`
and a `Clock`, and resolves *those* through `get()` recursively, which finds
their explicit bindings. This is roughly how Laravel's service container
and Symfony's DI container work under the hood, at far larger scale.

## PHP traps

**Reflection can't autowire scalar parameters.** `getType()` returns
`ReflectionNamedType` for `int $limit` too, but `isBuiltin()` is `true` for
scalars — the container above deliberately throws rather than guessing a
value, because there's no sane default for "what int should this be." Real
containers solve this with explicit binding of scalar/array parameters, or
by passing them at call time instead of relying on autowiring.

**`bind()` closures run lazily, but `get()` caches eagerly.** The factory
passed to `bind()` only runs the *first* time that abstract is requested;
after that, `instances[$abstract]` short-circuits `get()` and the same
object is handed out every time — that's why `$report === $report2` is
`true`. If you actually want a *new* instance per `get()` call (a common
need for request-scoped objects), you must not populate `$instances` at
all for that binding — this container conflates "singleton" and "how to
build," which a production container makes an explicit choice
(`singleton()` vs. `bind()`).

**A container is a tool for wiring, not a service locator to inject
everywhere.** Passing the `Container` itself into a class and calling
`$container->get(...)` from inside it hides the class's real dependencies
and defeats the entire point of declaring them in the constructor — you
lose the ability to see what a class needs just by reading its signature,
and testing gets harder, not easier. Inject the container only at the
composition root (where the application is bootstrapped), never into
domain classes.

## DI cheat sheet

| Concept | What it means |
|---|---|
| Constructor injection | Dependencies declared as typed constructor parameters |
| Depend on interfaces | Makes the concrete implementation swappable (real vs. fake) |
| Manual wiring | `new A(new B(), new C())` — fine for a handful of classes |
| Container | Centralizes "how to build X"; resolves dependency graphs |
| `bind($abstract, $factory)` | Explicit recipe for building an interface/class |
| Autowiring | Container inspects constructors via `Reflection` and resolves automatically |
| Service locator (anti-pattern) | Injecting the container itself and calling `->get()` inside a class |

## Exercise

Add a `NullLogger implements Logger` (a `log()` that does nothing) and a
`FixedClock implements Clock` whose constructor takes a `DateTimeImmutable`
and always returns it from `now()`. Bind both into a *second* `Container`
instance (`bind(Logger::class, fn() => new NullLogger())`,
`bind(Clock::class, fn() => new FixedClock(new DateTimeImmutable('2030-01-01')))`),
resolve a `ReportGenerator` from it, and confirm `generate()` returns
`"Report as of 2030-01-01"` with no `[LOG]` line printed — proving the same
`ReportGenerator` class works unmodified against completely different
collaborators, purely by changing what the container binds.
