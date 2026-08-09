# 02 · Design Patterns in PHP

A **design pattern** is a named, reusable solution to a recurring design
problem — not a library you install, but a shape of code you recognize and
apply by hand. Knowing the names matters less than recognizing *when* one
applies: they give you and your teammates a shared vocabulary ("just make it
a Strategy") instead of re-explaining the same structure every time it comes
up. This module walks through five patterns you'll actually run into in PHP
codebases, each with idiomatic PHP (`match`, constructor property promotion,
`interface`) rather than the boilerplate the original Gang-of-Four book
needed in a language without those features.

## Singleton — exactly one instance, globally accessible

```php
<?php
declare(strict_types=1);

final class Config
{
    private static ?Config $instance = null;
    private array $values = [];

    // Private constructor -- the ONLY way to get an instance is instance(),
    // so "new Config()" from outside the class is a compile-time error.
    private function __construct()
    {
        $this->values = ["env" => "production"];
    }

    public static function instance(): self
    {
        return self::$instance ??= new self();
    }

    public function get(string $key): ?string
    {
        return $this->values[$key] ?? null;
    }
}

$a = Config::instance();
$b = Config::instance();
var_dump($a === $b);              // bool(true) -- same object, not a copy
echo Config::instance()->get("env"), "\n";
```

```text
bool(true)
production
```

**The PHP-specific trap:** Singleton is the most overused pattern in the
book. It quietly turns into global mutable state, which makes unit tests
hard — every test that touches `Config::instance()` shares the *same*
object, so one test's changes can leak into the next. Prefer
[dependency injection](08-dependency-injection.md) (pass the config object
in) for anything you'll want to test in isolation; reach for Singleton only
for things that are genuinely single, process-wide, and read-only, like a
logger or app configuration.

## Factory Method — centralize object creation

```php
<?php
declare(strict_types=1);

interface Notification
{
    public function send(string $message): string;
}

final class EmailNotification implements Notification
{
    public function send(string $message): string
    {
        return "Email: $message";
    }
}

final class SmsNotification implements Notification
{
    public function send(string $message): string
    {
        return "SMS: $message";
    }
}

final class NotificationFactory
{
    public static function create(string $type): Notification
    {
        return match ($type) {
            "email" => new EmailNotification(),
            "sms" => new SmsNotification(),
            default => throw new InvalidArgumentException("Unknown notification type: $type"),
        };
    }
}

echo NotificationFactory::create("email")->send("Order shipped"), "\n";
echo NotificationFactory::create("sms")->send("Order shipped"), "\n";
```

```text
Email: Order shipped
SMS: Order shipped
```

Calling code depends only on the `Notification` interface and a string —
never on `EmailNotification` or `SmsNotification` directly. Add a
`PushNotification` class next month and every existing call site keeps
working unchanged, since the `match` in one place is the only spot that
needs to know the concrete classes exist.

## Strategy — swap an algorithm at runtime

```php
<?php
declare(strict_types=1);

interface DiscountStrategy
{
    public function apply(float $total): float;
}

final class NoDiscount implements DiscountStrategy
{
    public function apply(float $total): float
    {
        return $total;
    }
}

final class PercentOff implements DiscountStrategy
{
    public function __construct(private float $percent) {}

    public function apply(float $total): float
    {
        return $total * (1 - $this->percent / 100);
    }
}

final class Cart
{
    // The Cart doesn't know or care HOW the discount is calculated --
    // it just delegates to whatever strategy it was given.
    public function __construct(private DiscountStrategy $discount) {}

    public function checkoutTotal(float $subtotal): float
    {
        return $this->discount->apply($subtotal);
    }
}

$regular = new Cart(new NoDiscount());
$sale = new Cart(new PercentOff(20));

echo $regular->checkoutTotal(100.0), "\n";
echo $sale->checkoutTotal(100.0), "\n";
```

```text
100
80
```

This is the pattern behind PHPUnit's `createMock()` from [Level 2's testing
module](../level-2/05-phpunit-testing.md) too: swapping a real collaborator
for a fake one at construction time is Strategy in disguise, and it's the
same reason coding against an interface pays off for testability.

## Observer — notify subscribers without coupling to them

```php
<?php
declare(strict_types=1);

interface OrderObserver
{
    public function onOrderPlaced(string $orderId): void;
}

final class EmailReceiptObserver implements OrderObserver
{
    public function onOrderPlaced(string $orderId): void
    {
        echo "Emailing receipt for order $orderId\n";
    }
}

final class InventoryObserver implements OrderObserver
{
    public function onOrderPlaced(string $orderId): void
    {
        echo "Decrementing stock for order $orderId\n";
    }
}

final class OrderPlacer
{
    /** @var OrderObserver[] */
    private array $observers = [];

    public function subscribe(OrderObserver $observer): void
    {
        $this->observers[] = $observer;
    }

    public function place(string $orderId): void
    {
        echo "Order $orderId placed.\n";
        foreach ($this->observers as $observer) {
            $observer->onOrderPlaced($orderId);
        }
    }
}

$placer = new OrderPlacer();
$placer->subscribe(new EmailReceiptObserver());
$placer->subscribe(new InventoryObserver());
$placer->place("A1001");
```

```text
Order A1001 placed.
Emailing receipt for order A1001
Decrementing stock for order A1001
```

`OrderPlacer` never imports or references `EmailReceiptObserver` — new
side effects (an SMS alert, an analytics event) are added by writing a new
class and calling `subscribe()`, not by editing `place()`. This is the same
shape as PHP's built-in `SplSubjectInterface`/`SplObserver`, and the same
idea a queue's job-completed callback is often built on (see
[Working with Queues](06-working-with-queues.md)).

## Decorator — wrap an object to add behavior

```php
<?php
declare(strict_types=1);

interface Coffee
{
    public function cost(): float;
    public function description(): string;
}

final class PlainCoffee implements Coffee
{
    public function cost(): float { return 2.50; }
    public function description(): string { return "Coffee"; }
}

// Implements the SAME interface it wraps, so decorators can be stacked
// arbitrarily deep and calling code can't tell the difference.
abstract class CoffeeDecorator implements Coffee
{
    public function __construct(protected Coffee $coffee) {}
}

final class WithMilk extends CoffeeDecorator
{
    public function cost(): float { return $this->coffee->cost() + 0.50; }
    public function description(): string { return $this->coffee->description() . " + milk"; }
}

final class WithExtraShot extends CoffeeDecorator
{
    public function cost(): float { return $this->coffee->cost() + 0.75; }
    public function description(): string { return $this->coffee->description() . " + extra shot"; }
}

$order = new WithExtraShot(new WithMilk(new PlainCoffee()));
echo $order->description(), ": $", number_format($order->cost(), 2), "\n";
```

```text
Coffee + milk + extra shot: $3.75
```

Each decorator only knows about the one `Coffee` it wraps directly, and
delegates to it before adding its own cost — that's why wrapping order
matters (`WithMilk(WithExtraShot(...))` gives the same total but a
different `description()` order). This is exactly how [middleware
pipelines](07-middleware-patterns.md) work — each layer wraps the next and
adds behavior before/after delegating.

## Pattern cheat sheet

| Pattern | Problem it solves | PHP-idiomatic hook |
|---------|--------------------|--------------------|
| Singleton | Exactly one shared instance | Private constructor + `??=` in a static method |
| Factory Method | Centralize which concrete class gets built | `match` returning an interface type |
| Strategy | Swap an algorithm without changing the caller | Constructor-injected interface |
| Observer | Notify many listeners without coupling to them | An array of interface instances + `foreach` |
| Decorator | Add behavior by wrapping, not inheriting | Class implementing the same interface it wraps |

## Exercise

Add a `PushNotification` class to the Factory Method example and register
it as `"push"` in `NotificationFactory::create()` — confirm the existing
`"email"` and `"sms"` call sites keep working unchanged. Then write a
second `Coffee` decorator, `WithVanillaSyrup` (cost `+0.60`), and build an
order combining all three decorators in two different wrapping orders —
confirm `cost()` is identical either way but `description()` reads
differently, and explain why in a one-line comment.
