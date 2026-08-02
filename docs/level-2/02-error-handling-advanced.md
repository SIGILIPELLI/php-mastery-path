# 02 · Error Handling Advanced

[Level 1](../level-1/08-error-handling.md) covered `try`/`catch`/`finally`,
throwing built-in exceptions, and writing one custom exception class. In a
real application you need more: a whole **hierarchy** of related exceptions,
a way to preserve the original failure when you wrap it in a higher-level
one, and a global safety net for whatever slips through every `catch` block.

## Building an exception hierarchy

Instead of one flat custom exception, model your application's failure modes
as a small class tree rooted in a base type. Callers can then catch broadly
(the base type) or narrowly (a specific subtype) depending on how much detail
they need.

```php
<?php

// A base type for every error this application raises on purpose
abstract class AppException extends RuntimeException {}

class ValidationException extends AppException
{
    public function __construct(private array $errors)
    {
        parent::__construct("Validation failed: " . implode(", ", $errors));
    }

    public function getErrors(): array
    {
        return $this->errors;
    }
}

class NotFoundException extends AppException
{
    public function __construct(string $entity, int $id)
    {
        parent::__construct("$entity #$id not found");
    }
}

function findUser(int $id): array
{
    $users = [1 => ["id" => 1, "name" => "Ada"]];

    if (!isset($users[$id])) {
        throw new NotFoundException("User", $id);
    }
    return $users[$id];
}

try {
    findUser(99);
} catch (ValidationException $e) {
    echo "Bad input: " . implode(", ", $e->getErrors());
} catch (NotFoundException $e) {
    echo "Missing: " . $e->getMessage();
} catch (AppException $e) {
    // Safety net for any AppException subtype added later that isn't
    // handled above yet -- still narrower than catching every Throwable.
    echo "App error: " . $e->getMessage();
}
// Missing: User #99 not found
```

Marking `AppException` `abstract` stops anyone from throwing a bare,
meaningless `new AppException()` — every thrown exception must be one of the
specific subtypes, which keeps `catch` blocks meaningful.

## Exception chaining: preserving the original cause

When you catch a low-level exception and rethrow a more meaningful one, don't
discard the original — every built-in exception constructor accepts an
optional third `$previous` argument for exactly this.

```php
<?php

class DatabaseException extends RuntimeException {}
class ReportGenerationException extends RuntimeException {}

function queryDatabase(): never
{
    throw new DatabaseException("Connection refused on port 5432");
}

function generateReport(): void
{
    try {
        queryDatabase();
    } catch (DatabaseException $e) {
        // Wrap the low-level error in a message that makes sense at this
        // layer, but chain $e so nothing is lost.
        throw new ReportGenerationException(
            "Could not generate report",
            previous: $e,
        );
    }
}

try {
    generateReport();
} catch (ReportGenerationException $e) {
    echo $e->getMessage() . "\n";
    // Could not generate report
    echo "Caused by: " . $e->getPrevious()->getMessage() . "\n";
    // Caused by: Connection refused on port 5432
}
```

Without chaining, logs only ever show the high-level, user-facing message
("Could not generate report") and the real root cause is gone forever.
`getPrevious()` walks back one link at a time; PHP's default uncaught-error
output also prints the full chain automatically under "Next" sections.

## `finally` and `return`: a subtle gotcha

`finally` always runs, even after a `return` inside `try` or `catch` — but a
`return` *inside* `finally` silently overrides any value already being
returned. Avoid returning from `finally`; use it only for cleanup.

```php
<?php

function riskyRead(): string
{
    try {
        return "data";
    } finally {
        echo "cleanup ran\n";   // runs before the caller sees the return value
        // return "overridden";   // if uncommented, THIS becomes the return
        // value instead of "data" -- almost always a bug, not intentional
    }
}

echo riskyRead();
// cleanup ran
// data
```

## Converting PHP errors into exceptions

Not every failure in PHP is an exception by default — some legacy-style
functions raise warnings/notices instead. `set_error_handler()` lets you
convert those into `ErrorException`s so they flow through the same
`try`/`catch` machinery as everything else, instead of being easy to miss in
a log file.

```php
<?php

set_error_handler(function (int $severity, string $message, string $file, int $line) {
    // Converting to an exception means a stray warning can no longer be
    // silently ignored -- it now has to be caught (or it crashes loudly).
    throw new ErrorException($message, 0, $severity, $file, $line);
});

try {
    $value = $undefinedVariable;   // normally just a silent-ish warning
} catch (ErrorException $e) {
    echo "Caught as exception: " . $e->getMessage();
    // Caught as exception: Undefined variable $undefinedVariable
}

restore_error_handler();   // put PHP's default handler back
```

## A last-resort handler for uncaught exceptions

`set_exception_handler()` registers a function that runs if an exception
escapes every `try`/`catch` in your code. It cannot recover the program —
execution ends right after — but it's the right place to log the failure and
show a clean message instead of a raw stack trace.

```php
<?php

set_exception_handler(function (Throwable $e): void {
    error_log("Uncaught: " . $e->getMessage());
    echo "Something went wrong. Please try again later.\n";
});

function process(): void
{
    throw new RuntimeException("Unexpected failure deep in the call stack");
}

process();   // nothing catches this -- the handler above takes over
// Something went wrong. Please try again later.
```

In a real app this handler typically writes to a proper log (or an error
tracking service) rather than `error_log()`, and it never reveals internal
details like stack traces or database errors to the end user.

## Advanced error-handling cheat sheet

| Tool | Purpose |
|------|---------|
| `abstract class AppException extends RuntimeException` | Root of a custom exception hierarchy |
| `new Foo($msg, previous: $e)` | Chain the original exception so context isn't lost |
| `$e->getPrevious()` | Retrieve the chained (wrapped) exception, or `null` |
| `set_error_handler()` | Convert PHP warnings/notices into catchable `ErrorException`s |
| `set_exception_handler()` | Last-resort handler for anything left uncaught |
| `restore_error_handler()` | Undo a custom `set_error_handler()` registration |
| `finally` | Runs on every path out of `try`/`catch` — cleanup only, don't `return` here |

## Exercise

Design a small exception hierarchy for a file-processing task: an abstract
`FileProcessingException extends RuntimeException`, and two subclasses,
`FileNotReadableException` and `InvalidFormatException`. Write a function
`loadConfig(string $path): array` that throws `FileNotReadableException` if
`file_exists($path)` is false, and `InvalidFormatException` (chaining the
original `JsonException` as `$previous`) if `json_decode` with
`JSON_THROW_ON_ERROR` fails. Call it in a loop over a few paths (some
missing, some containing invalid JSON) and handle each subtype with its own
`catch` block, printing the chained previous message when there is one.
