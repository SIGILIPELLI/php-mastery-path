# 01 · Advanced Framework Patterns

Laravel, Symfony, and every other mature PHP framework are built from
patterns already covered in this path — [dependency injection](../level-3/08-dependency-injection.md),
[middleware](../level-3/07-middleware-patterns.md), a router — plus two more
that tie an application's bootstrap together: **service providers**, which
organize *what gets registered into the container*, and a lightweight
**ORM** (object-relational mapper) built on PHP's magic methods, which is
what `Product::find(1)->name` is doing under the hood in something like
Eloquent.

## Service providers: organizing container registration

A real application doesn't call `$container->bind(...)` inline all over the
place — each *feature area* (mail, database, queue) gets its own
**service provider** with two phases: `register()` (declare bindings — no
dependency resolution yet, since other providers may not have registered
theirs) and `boot()` (run once every provider has registered, safe to
resolve things now).

```php
<?php
// Container.php, ServiceProvider.php
declare(strict_types=1);

final class Container
{
    private array $bindings = [];
    private array $instances = [];

    public function bind(string $abstract, callable $factory): void
    {
        $this->bindings[$abstract] = $factory;
    }

    public function get(string $abstract): mixed
    {
        if (isset($this->instances[$abstract])) {
            return $this->instances[$abstract];
        }
        return $this->instances[$abstract] = ($this->bindings[$abstract])($this);
    }
}

interface ServiceProvider
{
    public function register(Container $c): void;
    public function boot(Container $c): void;
}
```

```php
<?php
// MailServiceProvider.php
declare(strict_types=1);

interface Mailer { public function send(string $to): void; }

final class SmtpMailer implements Mailer
{
    public function send(string $to): void { echo "Sending mail to $to via SMTP\n"; }
}

final class MailServiceProvider implements ServiceProvider
{
    public function register(Container $c): void
    {
        $c->bind(Mailer::class, fn() => new SmtpMailer());
        echo "MailServiceProvider: registered Mailer binding\n";
    }

    public function boot(Container $c): void
    {
        // Safe to RESOLVE here -- every provider's register() has already run.
        echo "MailServiceProvider: booted (mailer ready = " . get_class($c->get(Mailer::class)) . ")\n";
    }
}
```

```php
<?php
// Application.php
declare(strict_types=1);

final class Application
{
    private Container $container;
    /** @var ServiceProvider[] */
    private array $providers = [];

    public function __construct() { $this->container = new Container(); }

    public function register(ServiceProvider $p): void
    {
        $this->providers[] = $p;
        $p->register($this->container);
    }

    public function boot(): void
    {
        foreach ($this->providers as $p) {
            $p->boot($this->container);
        }
    }

    public function make(string $abstract): mixed { return $this->container->get($abstract); }
}
```

```php
<?php
// demo.php
require __DIR__ . '/Application.php';
require __DIR__ . '/MailServiceProvider.php';

$app = new Application();
$app->register(new MailServiceProvider());
$app->boot();
$app->make(Mailer::class)->send('ada@example.com');
```

```text
MailServiceProvider: registered Mailer binding
MailServiceProvider: booted (mailer ready = SmtpMailer)
Sending mail to ada@example.com via SMTP
```

The two-phase split matters the moment two providers depend on each other:
`QueueServiceProvider::boot()` might need the `Mailer` binding to already
exist to wire up a "send email" job handler. If registration and resolution
happened in a single pass, provider *order* would silently determine
whether the app boots — splitting into `register()` (declare only) then
`boot()` (resolve freely) removes that ordering dependency entirely.

## A minimal ORM: magic methods as the mechanism

`$product->name` reading and writing an internal `$attributes` array,
without `Product` declaring a `$name` property at all, is what makes
Eloquent/Doctrine-style models feel like plain objects. It's built on two
PHP magic methods: `__get()` and `__set()`, called automatically when code
touches an undeclared (or inaccessible) property.

```php
<?php
// Model.php
declare(strict_types=1);

abstract class Model
{
    protected static PDO $pdo;
    protected static string $table;
    protected array $attributes = [];

    public static function boot(PDO $pdo): void { static::$pdo = $pdo; }

    public function __get(string $name): mixed { return $this->attributes[$name] ?? null; }
    public function __set(string $name, mixed $value): void { $this->attributes[$name] = $value; }

    public function save(): void
    {
        $cols = array_keys($this->attributes);
        $placeholders = array_map(fn($c) => ":$c", $cols);
        $sql = sprintf(
            'INSERT INTO %s (%s) VALUES (%s)',
            static::$table, implode(',', $cols), implode(',', $placeholders)
        );
        $stmt = static::$pdo->prepare($sql);
        $stmt->execute($this->attributes);
        $this->attributes['id'] = (int) static::$pdo->lastInsertId();
    }

    public static function find(int $id): ?static
    {
        $stmt = static::$pdo->prepare('SELECT * FROM ' . static::$table . ' WHERE id = :id');
        $stmt->execute(['id' => $id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        if ($row === false) {
            return null;
        }
        $model = new static();
        $model->attributes = $row;
        return $model;
    }
}
```

```php
<?php
// Product.php + demo
declare(strict_types=1);
require __DIR__ . '/Model.php';

final class Product extends Model
{
    protected static string $table = 'products';
}

$pdo = new PDO('sqlite::memory:');
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$pdo->exec('CREATE TABLE products (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, price INTEGER)');
Product::boot($pdo);

$p = new Product();
$p->name = 'Keyboard';    // __set() -- no "name" property declared on Product
$p->price = 4999;
$p->save();
echo "Saved product id={$p->id}\n";   // __get()

$found = Product::find($p->id);
echo "Found: {$found->name} at {$found->price} cents\n";
```

```text
Saved product id=1
Found: Keyboard at 4999 cents
```

`static::$table` and `new static()` (not `self::`) are what let `find()` be
written once on `Model` and still return a `Product` — and every other
subclass's own table — correctly. `self::` would hard-code whichever class
*defined* the method; `static::` resolves against the class actually
*called*, PHP's "late static binding."

## PHP traps

**Magic methods only fire for inaccessible/undeclared properties.** If
`Product` accidentally declares `public string $name` directly, reading
`$p->name` uses that real property and `__get()` is never called — mixing
declared properties with magic ones on the same model is a common source of
"why isn't my accessor running" confusion. Pick one approach per class.

**`self::` vs. `static::` is invisible until you subclass.** A `Model`
method written with `self::$table` would always resolve to `Model`'s own
(non-existent) `$table`, not `Product::$table` — this only breaks the
moment a subclass actually calls the inherited method, which can be far
from where the bug was introduced.

**Service providers that resolve dependencies inside `register()`** (instead
of `boot()`) work by accident until provider order changes — resolving
`Mailer` from `register()` before every provider has registered its own
bindings is a race condition disguised as working code. Keep `register()`
strictly to declaring bindings.

## Advanced patterns cheat sheet

| Pattern | Purpose |
|---|---|
| Service provider | Groups related container bindings + boot logic per feature |
| `register()` | Declare bindings only — no resolving yet |
| `boot()` | Runs after all providers registered — safe to resolve |
| `__get()` / `__set()` | Intercept access to undeclared/inaccessible properties |
| `static::` (late static binding) | Resolves against the calling class, not the defining one |
| `new static()` | Instantiate the actual subclass, not the base class |

## Exercise

Add a `QueueServiceProvider` that binds a `Queue` interface (reuse the
`FileQueue` idea from [Working with Queues](../level-3/06-working-with-queues.md)
conceptually, or a simple in-memory array-backed stand-in) and, in its
`boot()`, resolves `Mailer` from the container and prints a line proving it
can see the binding `MailServiceProvider` registered. Register both
providers on one `Application`, call `boot()` once, and confirm the output
shows `MailServiceProvider` and `QueueServiceProvider` both booting
successfully regardless of registration order — register `QueueServiceProvider`
first and confirm it still works, since `boot()` for all providers only
runs after every `register()` has completed.
