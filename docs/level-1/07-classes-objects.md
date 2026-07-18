# 07 · Classes & Objects Basics

A class is a blueprint; an object is an instance created from that blueprint.
PHP's object model will feel familiar if you've used Java or JavaScript
classes, with a few PHP-specific conveniences layered on top.

## Properties and a basic constructor

```php
<?php

class Person
{
    // Properties (instance state). Typed properties are recommended in PHP 8.
    public string $name;
    public int $age;

    // Constructor -- runs automatically when you create a new Person
    public function __construct(string $name, int $age)
    {
        $this->name = $name;   // $this->name is the property, $name is the parameter
        $this->age = $age;
    }

    public function introduce(): string
    {
        return "Hi, I'm {$this->name}, age {$this->age}";
    }
}

$alice = new Person("Alice", 30);
$bob = new Person("Bob", 25);

echo $alice->introduce();   // Hi, I'm Alice, age 30
echo $bob->introduce();     // Hi, I'm Bob, age 25

echo $alice->name;   // Alice -- direct property access via ->
```

`$this` refers to the current instance — it's how a method or constructor
refers to "this particular object's" data, distinguishing a property from a
same-named parameter.

## Constructor property promotion (PHP 8+)

Writing `public string $name;` plus a constructor parameter plus a `$this->name
= $name;` assignment is repetitive. PHP 8 lets you declare and assign
properties directly in the constructor signature:

```php
<?php

class Rectangle
{
    // Promoted properties -- this single line declares $width and $height
    // as public properties AND assigns them from the constructor arguments.
    public function __construct(
        public float $width,
        public float $height,
    ) {}

    public function area(): float
    {
        return $this->width * $this->height;
    }
}

$r = new Rectangle(width: 4, height: 5);   // named arguments, PHP 8+
echo $r->area();   // 20

// A square built by delegating to the same constructor
class Square extends Rectangle
{
    public function __construct(float $side)
    {
        parent::__construct($side, $side);
    }
}

$square = new Square(3);
echo $square->area();   // 9
```

Constructor promotion is now the idiomatic way to write simple data-holding
classes in modern PHP — it removes a lot of boilerplate.

## Visibility: public, private, protected

```php
<?php

class BankAccount
{
    // private -- only accessible from inside this class
    private float $balance;

    public function __construct(float $initialBalance)
    {
        if ($initialBalance < 0) {
            throw new InvalidArgumentException("Initial balance cannot be negative");
        }
        $this->balance = $initialBalance;
    }

    public function getBalance(): float
    {
        return $this->balance;
    }

    public function deposit(float $amount): void
    {
        if ($amount <= 0) {
            throw new InvalidArgumentException("Deposit must be positive");
        }
        $this->balance += $amount;
    }

    public function withdraw(float $amount): void
    {
        if ($amount > $this->balance) {
            throw new RuntimeException("Insufficient funds");
        }
        $this->balance -= $amount;
    }
}

$account = new BankAccount(100.0);
$account->deposit(50.0);
$account->withdraw(30.0);
echo $account->getBalance();   // 120

// echo $account->balance;   // Error: Cannot access private property
```

| Visibility | Accessible from |
|------------|------------------|
| `public` | Anywhere |
| `protected` | This class and subclasses (not from outside) |
| `private` | Only this exact class |

Making a property `private` (or `protected`) and exposing controlled access
through methods like `getBalance()` and `deposit()` is **encapsulation** — it
stops external code from putting the object into an invalid state.

## Getters/setters with validation

```php
<?php

class User
{
    private string $email;

    public function __construct(string $email)
    {
        $this->setEmail($email);   // reuse validation in the constructor too
    }

    public function getEmail(): string
    {
        return $this->email;
    }

    public function setEmail(string $email): void
    {
        if (!str_contains($email, "@")) {
            throw new InvalidArgumentException("Invalid email: $email");
        }
        $this->email = $email;
    }
}

$user = new User("alice@example.com");
echo $user->getEmail();   // alice@example.com

$user->setEmail("bob@example.com");
echo $user->getEmail();   // bob@example.com

try {
    $user->setEmail("not-an-email");
} catch (InvalidArgumentException $e) {
    echo "Rejected: " . $e->getMessage();
    // Rejected: Invalid email: not-an-email
}
```

## Static properties and methods

Static members belong to the *class itself*, not to any one instance — useful
for counters, factory methods, or utility functions that don't need object
state.

```php
<?php

class Counter
{
    private static int $count = 0;   // shared across all instances

    public function __construct()
    {
        self::$count++;   // "self::" refers to the class, not an instance
    }

    public static function getCount(): int
    {
        return self::$count;
    }
}

new Counter();
new Counter();
new Counter();

echo Counter::getCount();   // 3 -- called on the class, no object needed
```

## The `->` operator and instantiation with `new`

```php
<?php

class Circle
{
    public function __construct(private float $radius) {}

    public function area(): float
    {
        return M_PI * $this->radius ** 2;
    }

    public function circumference(): float
    {
        return 2 * M_PI * $this->radius;
    }
}

$c1 = new Circle(2.0);
$c2 = new Circle(5.0);

printf("%.2f\n", $c1->area());   // 12.57 -- each object has its own radius
printf("%.2f\n", $c2->area());   // 78.54
```

`new ClassName(...)` creates an instance; `->` accesses properties and methods
on that instance. `::` (the "scope resolution operator") is used for static
members, class constants, and `parent::`/`self::` calls.

## Classes cheat sheet

| Concept | Meaning |
|---------|---------|
| Class | Blueprint / type definition |
| Object (instance) | A concrete value created with `new` |
| Property | A variable that lives on each instance (or on the class, if `static`) |
| Constructor (`__construct`) | Special method that initializes a new instance |
| `$this` | Reference to the current instance |
| `self::` | Reference to the current class (for static members) |
| `public` / `protected` / `private` | Visibility, from most to least accessible |
| `->` | Access a property/method on an instance |
| `::` | Access a static member, class constant, or parent method |

## Exercise

Write a `Book` class with private properties `title`, `author`, and
`pagesRead` (starting at `0`), a constructor taking `title` and `author`
(consider using constructor property promotion for the two string
properties), a method `readPages(int $n): void` that increases `pagesRead`,
and a method `getProgress(int $totalPages): float` that returns the
percentage read. Create two `Book` objects, call `readPages` on each with
different values, and print each one's progress with `printf`.
