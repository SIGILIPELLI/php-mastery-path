# 01 · OOP Deep Dive

[Level 1](../level-1/07-classes-objects.md) covered classes, properties, and
visibility. Real applications need more than single classes with no shared
contract — they need ways to guarantee that unrelated classes support the
same operations (**interfaces**), share partial implementations across a
family of related classes (**abstract classes**), and reuse method bodies
across classes that aren't related by inheritance at all (**traits**). These
three tools are how PHP does polymorphism and code reuse beyond simple
`extends`.

## Interfaces: contracts without implementation

An interface declares *what* methods a class must have, not *how* they work.
Any class that implements the interface must provide bodies for every method.

```php
<?php

interface Shape
{
    public function area(): float;
    public function perimeter(): float;
}

class Rectangle implements Shape
{
    public function __construct(private float $width, private float $height) {}

    public function area(): float
    {
        return $this->width * $this->height;
    }

    public function perimeter(): float
    {
        return 2 * ($this->width + $this->height);
    }
}

class Circle implements Shape
{
    public function __construct(private float $radius) {}

    public function area(): float
    {
        return M_PI * $this->radius ** 2;
    }

    public function perimeter(): float
    {
        return 2 * M_PI * $this->radius;
    }
}

// Works with ANY Shape -- doesn't care which concrete class it receives
function describe(Shape $shape): string
{
    return sprintf("area=%.2f perimeter=%.2f", $shape->area(), $shape->perimeter());
}

echo describe(new Rectangle(4, 5));   // area=20.00 perimeter=18.00
echo "\n";
echo describe(new Circle(3));         // area=28.27 perimeter=18.85
```

`Rectangle` and `Circle` share no parent class and store completely different
data, but because both implement `Shape`, `describe()` can accept either.
This is **polymorphism**: code written against the interface works with any
class that honors the contract, including ones that don't exist yet.

A class can implement several interfaces at once (separated by commas),
unlike `extends`, which only allows one parent class. Interfaces can also
extend other interfaces, and declare constants — but never properties or
method bodies. Checking whether an object satisfies an interface (or class)
at runtime uses `instanceof`:

```php
<?php
$r = new Rectangle(4, 5);
var_dump($r instanceof Shape);       // bool(true)
var_dump($r instanceof Rectangle);   // bool(true) -- also true for the concrete class
```

## Abstract classes: shared implementation plus required overrides

An abstract class is a base class that **cannot be instantiated directly**.
It can mix concrete (fully implemented) methods with `abstract` methods that
every subclass must implement. Use it when related classes share real code,
not just a contract.

```php
<?php

abstract class Employee
{
    public function __construct(
        protected string $name,
        protected float $baseSalary,
    ) {}

    // Concrete method -- shared by every subclass as-is
    public function getName(): string
    {
        return $this->name;
    }

    // Abstract method -- no body here; each subclass MUST provide one
    abstract public function calculatePay(): float;

    // Concrete method that calls the abstract one -- this is the
    // "template method" pattern: the algorithm's shape lives in the
    // base class, the varying step is delegated to subclasses.
    public function paySlip(): string
    {
        return sprintf("%s: $%.2f", $this->name, $this->calculatePay());
    }
}

class SalariedEmployee extends Employee
{
    public function calculatePay(): float
    {
        return $this->baseSalary / 12;   // annual salary paid monthly
    }
}

class CommissionEmployee extends Employee
{
    public function __construct(
        string $name,
        float $baseSalary,
        private float $commission,
    ) {
        parent::__construct($name, $baseSalary);
    }

    public function calculatePay(): float
    {
        return $this->baseSalary / 12 + $this->commission;
    }
}

$staff = [
    new SalariedEmployee("Priya", 84000),
    new CommissionEmployee("Marco", 60000, 1200),
];

foreach ($staff as $employee) {
    echo $employee->paySlip() . "\n";
}
// Priya: $7000.00
// Marco: $6200.00

// new Employee("X", 1000);   // Fatal error: Cannot instantiate abstract class
```

A subclass that forgets to implement an abstract method is caught at
class-definition time, not silently ignored: `class Incomplete extends
Employee {}` fails immediately with `Fatal error: Class Incomplete contains
1 abstract method and must therefore be declared abstract or implement the
remaining methods`.

## Traits: sharing method bodies across unrelated classes

Interfaces share a contract; abstract classes share a contract *and* an
inheritance relationship. Sometimes you want to share actual method bodies
between classes that have no business relationship at all — a `Logger` mixin
usable by both a `Payment` class and a `Report` class, for example. PHP only
allows single inheritance (`extends` one class), so traits fill that gap.

```php
<?php

trait Loggable
{
    private array $log = [];

    public function logMessage(string $message): void
    {
        $this->log[] = sprintf("[%s] %s", date("H:i:s"), $message);
    }

    public function getLog(): array
    {
        return $this->log;
    }
}

class Payment
{
    use Loggable;   // "copies in" logMessage() and getLog() as if written here

    public function process(float $amount): void
    {
        $this->logMessage("Processing payment of $amount");
    }
}

class Report
{
    use Loggable;

    public function generate(): void
    {
        $this->logMessage("Report generated");
    }
}

$payment = new Payment();
$payment->process(49.99);
print_r($payment->getLog());
// Array ( [0] => [10:22:01] Processing payment of 49.99 )

$report = new Report();
$report->generate();
print_r($report->getLog());
// Array ( [0] => [10:22:01] Report generated )
```

`Payment` and `Report` share no parent/child relationship — each got its own
independent `$log` array and its own copy of the trait's methods. A trait is
best thought of as "copy-paste that the language does for you," not a real
`is-a` relationship (unlike interfaces or abstract classes, `instanceof`
does **not** work against a trait name).

### Resolving conflicts between traits

If a class uses two traits that define the same method name, PHP requires
you to resolve the collision explicitly rather than silently picking one:

```php
<?php

trait Swimmer
{
    public function move(): string
    {
        return "swims";
    }
}

trait Runner
{
    public function move(): string
    {
        return "runs";
    }
}

class Duck
{
    use Swimmer, Runner {
        Swimmer::move insteadof Runner;   // prefer Swimmer's version for move()
        Runner::move as run;              // but keep Runner's version too, renamed
    }
}

$duck = new Duck();
echo $duck->move();   // swims
echo "\n";
echo $duck->run();    // runs
```

Without the `insteadof`/`as` block, combining two traits that define the same
method is a fatal error — PHP refuses to guess which one you meant.

## Choosing between them

| Tool | Instantiable? | Shares state? | Multiple at once? | Use when |
|------|:---:|:---:|:---:|----------|
| Interface | No | No (contract only) | Yes, unlimited | You need a guaranteed set of methods, no shared code |
| Abstract class | No | Yes | No, single `extends` | Related classes share real implementation plus required overrides |
| Trait | No | Yes (per including class) | Yes, unlimited | Unrelated classes need the same method bodies (no `is-a` relationship) |

A single class often combines all three: `implements` one or more
interfaces, `extends` at most one (possibly abstract) parent, and `use`s any
number of traits.

## Exercise

Define an interface `Notifiable` with one method, `notify(string $message):
void`. Create an abstract class `BaseNotifier implements Notifiable` with a
concrete `formatMessage(string $message): string` method that adds a
timestamp prefix, plus an abstract `send(string $formatted): void` — have
`notify()` call `formatMessage()` then `send()` (the template method
pattern). Write two concrete subclasses, `EmailNotifier` and `SmsNotifier`,
each implementing `send()` by just `echo`-ing what it would send. Then write
a trait `RateLimited` with a `hasSentTooRecently(): bool` method (track a
last-sent timestamp) and mix it into one of the notifiers to skip sending if
called twice within one second.
