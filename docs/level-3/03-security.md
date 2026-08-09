# 03 · Security

Level 2 already introduced two essential defenses — escaping output to
prevent [XSS](../level-2/04-sessions-forms.md#always-escape-output-preventing-xss)
and [CSRF tokens](../level-2/04-sessions-forms.md#csrf-making-sure-the-request-really-came-from-your-form) —
and [PDO with bound parameters](../level-2/03-databases-pdo.md) to prevent
SQL injection. This module goes deeper into PHP-specific attack surfaces
that don't come up until an app handles real users, real logins, and real
untrusted input: safe identifier handling, password storage, timing
attacks, insecure deserialization, and response-level hardening.

## SQL injection: the part prepared statements don't cover

Bound parameters neutralize injection in **values** (`WHERE id = :id`), but
a placeholder can never stand in for an **identifier** — a column or table
name — because PDO always renders parameters as quoted values, not raw SQL.
An `ORDER BY` clause built from user input needs a different defense: an
allowlist.

```php
<?php
declare(strict_types=1);

// Allowlisting: a column name can never be safely bound as a prepared
// statement parameter, so identifiers must be checked against a fixed
// list of known-good values instead.
function orderBy(string $column, array $allowed = ["name", "created_at", "price"]): string
{
    if (!in_array($column, $allowed, true)) {
        throw new InvalidArgumentException("Invalid sort column: $column");
    }
    return $column;
}

echo orderBy("price"), "\n";

try {
    orderBy("price; DROP TABLE users; --");
} catch (InvalidArgumentException $e) {
    echo "Blocked: ", $e->getMessage(), "\n";
}
```

```text
price
Blocked: Invalid sort column: price; DROP TABLE users; --
```

The same rule applies to table names, `LIMIT`/`OFFSET` when they come from
a non-numeric source, and dynamic column lists in an `INSERT` — anywhere
user input shapes the SQL text itself rather than a value inside it.

## Password storage: `password_hash()` and `password_verify()`

Never store a password in plain text, and never hash it with a fast,
general-purpose function like `md5()` or `sha256()` — those are *designed*
to be fast, which makes brute-forcing billions of guesses per second
trivial on modern hardware. `password_hash()` uses a slow, purpose-built
algorithm (bcrypt by default) and embeds a random salt automatically.

```php
<?php
declare(strict_types=1);

$hash1 = password_hash("correct horse battery staple", PASSWORD_DEFAULT);
$hash2 = password_hash("correct horse battery staple", PASSWORD_DEFAULT);

echo "Same password, different hashes: ", ($hash1 !== $hash2 ? "yes" : "no"), "\n";
echo "Hash prefix: ", substr($hash1, 0, 7), "\n";
echo "Verify correct password: ", (password_verify("correct horse battery staple", $hash1) ? "true" : "false"), "\n";
echo "Verify wrong password: ", (password_verify("wrong password", $hash1) ? "true" : "false"), "\n";

// needs_rehash lets you raise the cost factor over time without a mass
// password reset -- check it right after a successful login.
$oldHash = password_hash("legacy user", PASSWORD_BCRYPT, ["cost" => 4]);
echo "Needs rehash at cost 4 -> 12: ", (password_needs_rehash($oldHash, PASSWORD_BCRYPT, ["cost" => 12]) ? "true" : "false"), "\n";
```

```text
Same password, different hashes: yes
Hash prefix: $2y$12$
Verify correct password: true
Verify wrong password: false
Needs rehash at cost 4 -> 12: true
```

The hash string is self-describing — algorithm, cost, and salt all travel
inside it — so `password_verify()` never needs a separate lookup for "which
algorithm did we use for this user."

## Timing attacks: why `hash_equals()` exists

A plain `===` string comparison returns as soon as it finds the first
mismatched byte. That means comparing `"a..."` against a secret starting
with `"b..."` returns *microseconds* faster than comparing it against a
secret that also starts with `"a..."` — and an attacker who can measure
response times precisely enough can recover a secret token one byte at a
time. `hash_equals()` always takes the same amount of time regardless of
where (or whether) the strings differ.

```php
<?php
declare(strict_types=1);

function checkApiToken(string $provided, string $expected): bool
{
    return hash_equals($expected, $provided);
}

$expected = "sk_live_9f8a7b6c5d4e3f2a1b0c";

var_dump(checkApiToken("sk_live_9f8a7b6c5d4e3f2a1b0c", $expected));
var_dump(checkApiToken("wrong-token", $expected));
var_dump(checkApiToken("sk_live_9f8a7b6c5d4e3f2a1b0X", $expected));
```

```text
bool(true)
bool(false)
bool(false)
```

Use `hash_equals()` (never `===`) for API tokens, CSRF tokens, HMAC
signatures, and password reset tokens — anywhere a secret value is compared
against attacker-supplied input.

## Insecure deserialization: `unserialize()` on untrusted input

`unserialize()` rebuilds full PHP objects, including running their
constructors, `__wakeup()`, and eventually their destructors. If an
attacker can get arbitrary bytes into `unserialize()` (a cookie value, a
cache entry, a query parameter), and *any* class reachable in your app has
a "dangerous" magic method, that's a working exploit — known as **PHP
object injection**.

```php
<?php
declare(strict_types=1);

// Stands in for a real gadget chain -- a class that does something
// consequential (deletes a file, runs a command) as a side effect of
// being destroyed. In a real attack this is a class that already exists
// somewhere in your app or its dependencies, not one the attacker writes.
class FileCleanup
{
    public function __construct(private string $path) {}

    public function __destruct()
    {
        echo "FileCleanup ran for: {$this->path}\n";
    }
}

class SafeData
{
    public function __construct(public string $value) {}
}

$original = new FileCleanup("/var/www/important.txt");
$attackerInput = serialize($original);
unset($original); // this destructor firing here is just normal setup

$trustedInput = serialize(new SafeData("hello"));

echo "-- default unserialize(), no options (UNSAFE) --\n";
$unsafe = unserialize($attackerInput);
var_dump($unsafe instanceof FileCleanup);
unset($unsafe); // __destruct fires again here -- attacker's class just ran

echo "-- unserialize() restricted to an allowlist (SAFE) --\n";
$safe = unserialize($attackerInput, ["allowed_classes" => [SafeData::class]]);
var_dump($safe instanceof FileCleanup);   // false: downgraded, destructor never runs
unset($safe);

$safeData = unserialize($trustedInput, ["allowed_classes" => [SafeData::class]]);
var_dump($safeData instanceof SafeData);
echo $safeData->value, "\n";
```

```text
FileCleanup ran for: /var/www/important.txt
-- default unserialize(), no options (UNSAFE) --
bool(true)
FileCleanup ran for: /var/www/important.txt
-- unserialize() restricted to an allowlist (SAFE) --
bool(false)
bool(true)
hello
```

The first "FileCleanup ran" line is just the `unset($original)` cleanup
from setup. The **second** one is the exploit: `unserialize()` with no
options rebuilt a real `FileCleanup` object from attacker-controlled bytes
and ran its destructor. The allowlisted call downgrades any class not on
the list to a harmless `__PHP_Incomplete_Class` — no constructor,
destructor, or magic method from the original class ever executes. Prefer
`json_decode()` over `unserialize()` for anything crossing a trust boundary
(cookies, cache, queues) — JSON carries data, never behavior.

## Response hardening: security headers and cookie flags

```php
<?php
declare(strict_types=1);

function sendSecurityHeaders(): void
{
    header("X-Content-Type-Options: nosniff");              // stop MIME-sniffing away from the declared Content-Type
    header("X-Frame-Options: DENY");                          // stop the page being framed (clickjacking)
    header("Content-Security-Policy: default-src 'self'");    // restrict where scripts/styles/images may load from
    header("Referrer-Policy: strict-origin-when-cross-origin");
}

sendSecurityHeaders();
echo "Headers queued (visible via curl -I against a running server, not CLI)\n";
```

```text
Headers queued (visible via curl -I against a running server, not CLI)
```

Cookie flags matter as much as the headers around them — set before
`session_start()`:

```php
<?php
declare(strict_types=1);

session_set_cookie_params([
    "httponly" => true,   // JavaScript (document.cookie) can never read this cookie
    "samesite" => "Lax",  // browser withholds it on most cross-site requests (CSRF defense-in-depth)
    "secure" => true,     // only sent over HTTPS
]);

$params = session_get_cookie_params();
echo "httponly=" . var_export($params["httponly"], true)
    . " samesite={$params['samesite']}"
    . " secure=" . var_export($params["secure"], true) . "\n";
```

```text
httponly=true samesite=Lax secure=true
```

## Security cheat sheet

| Risk | Defense | PHP function/setting |
|------|---------|-----------------------|
| SQL injection (values) | Bound parameters | `PDO::prepare()` + `execute([...])` |
| SQL injection (identifiers) | Allowlist, never bind | `in_array($col, $allowed, true)` |
| Stored XSS | Escape on output, not input | `htmlspecialchars()` |
| CSRF | Per-session token, checked on every write | `hash_equals($session, $submitted)` |
| Weak password storage | Slow, salted, adaptive hash | `password_hash()` / `password_verify()` |
| Timing attack on secrets | Constant-time comparison | `hash_equals()` |
| PHP object injection | Never unserialize untrusted bytes | `unserialize($x, ["allowed_classes" => [...]])` or `json_decode()` |
| Clickjacking / MIME sniffing | Restrictive response headers | `header("X-Frame-Options: DENY")` |
| Session/cookie theft | Restrict cookie scope and transport | `session_set_cookie_params([...])` |

## Exercise

Write a function `validateSortColumn(string $column): string` that
allowlists `["title", "created_at", "id"]` (throwing `InvalidArgumentException`
otherwise), then a `checkAdminToken(string $submitted): bool` that compares
against a hard-coded expected token using `hash_equals()`. Write a small
script that calls both with one valid and one malicious input each, and
`echo`s a clear pass/fail line per case. Separately, take the CRUD
project's `Auth` class from [Level 2's capstone](../level-2/10-project-crud-app.md)
and add `session_set_cookie_params()` with `httponly`, `samesite`, and
`secure` flags before its `session_start()` call.
