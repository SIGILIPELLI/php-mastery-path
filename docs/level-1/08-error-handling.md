# 08 · Error Handling Basics

Things go wrong at runtime — bad input, missing files, division by zero.
PHP's exception system lets you detect problems and handle them in one place
instead of checking a return value after every risky operation.

## try / catch / finally

```php
<?php

function divide(int $a, int $b): float
{
    if ($b === 0) {
        throw new DivisionByZeroError("Cannot divide by zero");
    }
    return $a / $b;
}

try {
    echo divide(10, 2);   // 5
    echo divide(10, 0);   // never reached -- throws first
} catch (DivisionByZeroError $e) {
    echo "Error: " . $e->getMessage();
    // Error: Cannot divide by zero
} finally {
    echo "\nDone attempting division.";   // always runs, error or not
}
```

- The `try` block contains code that might fail.
- `catch` runs only if a matching exception type is thrown.
- `finally` always runs — whether or not an exception was thrown — and is the
  right place for cleanup (closing files, releasing resources).

## Throwing your own exceptions

```php
<?php

function withdraw(float $balance, float $amount): float
{
    if ($amount <= 0) {
        throw new InvalidArgumentException("Amount must be positive");
    }
    if ($amount > $balance) {
        throw new RuntimeException("Insufficient funds");
    }
    return $balance - $amount;
}

try {
    $newBalance = withdraw(100, 150);
} catch (RuntimeException $e) {
    echo $e->getMessage();   // Insufficient funds
}
```

`throw new SomeException("message")` immediately stops normal execution and
unwinds the call stack until a matching `catch` is found (or the script dies
with an uncaught error if none is).

## Catching specific exception types

Order matters: catch the most specific types first, since PHP matches the
first `catch` block whose type fits.

```php
<?php

function parseAge(string $input): int
{
    if (!is_numeric($input)) {
        throw new InvalidArgumentException("'$input' is not a number");
    }
    $age = (int) $input;
    if ($age < 0 || $age > 150) {
        throw new OutOfRangeException("$age is not a plausible age");
    }
    return $age;
}

foreach (["25", "abc", "-5", "999"] as $input) {
    try {
        $age = parseAge($input);
        echo "Parsed age: $age\n";
    } catch (InvalidArgumentException $e) {
        // Note: OutOfRangeException is NOT a subclass of InvalidArgumentException,
        // so this only catches the "not a number" case.
        echo "Invalid input: {$e->getMessage()}\n";
    } catch (OutOfRangeException $e) {
        echo "Out of range: {$e->getMessage()}\n";
    }
}
// Parsed age: 25
// Invalid input: 'abc' is not a number
// Out of range: -5 is not a plausible age
// Out of range: 999 is not a plausible age
```

## Multi-catch: handling several types the same way

When two or more exception types should be handled identically, combine them
with `|` in a single `catch`:

```php
<?php

function riskyOperation(int $mode): string
{
    return match ($mode) {
        1 => throw new InvalidArgumentException("bad argument"),
        2 => throw new RuntimeException("bad runtime state"),
        default => "ok",
    };
};

foreach ([1, 2, 3] as $mode) {
    try {
        echo riskyOperation($mode) . "\n";
    } catch (InvalidArgumentException | RuntimeException $e) {
        // Same handling for either type -- log and move on
        echo "Handled: " . get_class($e) . " - " . $e->getMessage() . "\n";
    }
}
// Handled: InvalidArgumentException - bad argument
// Handled: RuntimeException - bad runtime state
// ok
```

## Built-in exception classes

PHP ships a family of SPL exception classes for common failure categories —
prefer these over a bare `Exception` so callers can catch precisely:

| Class | Typical use |
|-------|-------------|
| `InvalidArgumentException` | A function argument is the wrong type/value |
| `OutOfRangeException` | An illegal index or value outside an expected range |
| `RuntimeException` | An error that can only be detected while running (e.g. failed I/O) |
| `LogicException` | A programming error that should have been caught before running |
| `LengthException` | A length is invalid (e.g. empty required string) |
| `DivisionByZeroError` | Division or modulo by zero (this one is an `Error`, not `Exception`) |
| `TypeError` | A value doesn't match a declared type (an `Error`, not `Exception`) |

## Errors vs. Exceptions: the `Throwable` interface

In PHP 8, both `Exception` and `Error` implement a common interface,
`Throwable`. Historically, `Exception` represented recoverable application
conditions (bad input, failed validation) while `Error` represents things the
engine itself detects (type mismatches, calling a method on null, division by
zero) — closer to what other languages call a "fatal error." Since PHP 7,
`Error` is also catchable, and both can be caught the same way.

```php
<?php

function process(mixed $value): int
{
    // Calling strlen() on something that isn't a string triggers a TypeError
    return strlen($value);
}

try {
    process(null);   // TypeError, not Exception, in PHP 8
} catch (Throwable $e) {
    // Catching Throwable is a safety net that covers BOTH Error and Exception
    echo get_class($e) . ": " . $e->getMessage() . "\n";
}
```

Prefer catching specific `Exception` subclasses in application logic; reserve
catching `Throwable` for top-level safety nets (like a global request
handler) where you must never let anything escape uncaught.

## Custom exception classes

For domain-specific errors, extend `Exception` (or a more specific built-in
subclass) so callers can catch exactly your error type and you can attach
extra context.

```php
<?php

class InsufficientFundsException extends RuntimeException
{
    public function __construct(
        private float $requested,
        private float $available,
    ) {
        parent::__construct(
            sprintf("Requested %.2f but only %.2f available", $requested, $available)
        );
    }

    public function getShortfall(): float
    {
        return $this->requested - $this->available;
    }
}

function withdraw(float $balance, float $amount): float
{
    if ($amount > $balance) {
        throw new InsufficientFundsException($amount, $balance);
    }
    return $balance - $amount;
}

try {
    withdraw(50, 200);
} catch (InsufficientFundsException $e) {
    echo $e->getMessage() . "\n";
    // Requested 200.00 but only 50.00 available
    echo "Short by: " . $e->getShortfall();
    // Short by: 150
}
```

Custom exceptions can carry structured data (like `getShortfall()` above) that
a generic `Exception` message string can't — callers get both a human-readable
message and machine-usable details.

## Error handling cheat sheet

| Keyword / class | Purpose |
|------------------|---------|
| `try { ... }` | Wrap code that might throw |
| `catch (Type $e) { ... }` | Handle a specific exception/error type |
| `catch (TypeA \| TypeB $e) { ... }` | Handle either of two types the same way |
| `finally { ... }` | Always runs, for cleanup |
| `throw new Exception("msg")` | Raise an exception |
| `$e->getMessage()` | Retrieve the error message |
| `$e->getCode()` | Retrieve an optional numeric error code |
| `Throwable` | Interface implemented by both `Exception` and `Error` |

## Exercise

Write a custom exception class `NegativeDepositException extends
InvalidArgumentException` that stores the invalid amount and exposes a
`getAmount(): float` method. Then write a function `deposit(float $balance,
float $amount): float` that throws `NegativeDepositException` when `$amount`
is less than or equal to `0`, and otherwise returns the new balance. Call it
in a loop with a few good and bad amounts, catching your custom exception and
printing a friendly message that includes the rejected amount.
