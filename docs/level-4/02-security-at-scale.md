# 02 · Security at Scale

[Level 3's security module](../level-3/03-security.md) covered SQL
injection, XSS, and password hashing for a single application. At scale —
multiple environments, multiple developers, secrets that must never touch
version control — the concerns shift toward *process*: where secrets live,
how requests are proven genuine, and what a hardened production
configuration actually looks like.

## Secrets don't belong in source code

A database password hard-coded in a config file that gets committed to git
is a leak the moment the repository is cloned by anyone, or made public by
accident. The standard fix is an **environment-based** secrets store: values
live outside the codebase (a `.env` file, excluded from git, or actual
process environment variables set by the deploy platform) and application
code reads them through a single access point.

```php
<?php
// EnvSecrets.php
declare(strict_types=1);

final class EnvSecrets
{
    private array $values = [];

    public function __construct(string $path)
    {
        if (!is_file($path)) {
            return;
        }
        foreach (file($path, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES) as $line) {
            if (str_starts_with(trim($line), '#')) {
                continue;
            }
            [$key, $value] = explode('=', $line, 2);
            $this->values[trim($key)] = trim($value);
        }
    }

    public function get(string $key, ?string $default = null): ?string
    {
        return $this->values[$key] ?? getenv($key) ?: $default;
    }

    public function require(string $key): string
    {
        $v = $this->get($key);
        if ($v === null) {
            throw new RuntimeException("Missing required secret: $key");
        }
        return $v;
    }
}
```

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/EnvSecrets.php';

file_put_contents('/tmp/test.env', "DB_PASSWORD=s3cret\n# comment\nAPI_KEY=abc123\n");

$secrets = new EnvSecrets('/tmp/test.env');
echo $secrets->require('DB_PASSWORD') . "\n";
echo $secrets->get('MISSING_KEY', 'fallback') . "\n";

try {
    $secrets->require('DOES_NOT_EXIST');
} catch (RuntimeException $e) {
    echo "Caught: " . $e->getMessage() . "\n";
}
```

```text
s3cret
fallback
Caught: Missing required secret: DOES_NOT_EXIST
```

`require()` failing loudly and immediately — rather than the app limping
along with a `null` database password until a connection fails somewhere
deep in a request — is deliberate: a missing secret should be a startup
error, not a runtime mystery three layers down. `.env` files are for local
development; in real production deployments the equivalent role is usually
played by the platform's own secret store (a managed secrets manager,
CI/CD-injected environment variables) so the value never sits in a file on
disk at all.

## CSRF protection: proving the request came from your own form

A **cross-site request forgery** tricks a logged-in user's browser into
submitting a request to your site from a *different* site — the browser
automatically attaches the user's cookies, so the request looks
authenticated even though the user never intended it. The fix is a random,
unguessable **token** embedded in your own forms, checked against the one
stored in the user's session.

```php
<?php
// Csrf.php
declare(strict_types=1);

final class Csrf
{
    public static function token(array &$session): string
    {
        if (!isset($session['csrf_token'])) {
            $session['csrf_token'] = bin2hex(random_bytes(32));
        }
        return $session['csrf_token'];
    }

    public static function verify(array $session, string $submitted): bool
    {
        $expected = $session['csrf_token'] ?? '';
        return $expected !== '' && hash_equals($expected, $submitted);
    }
}
```

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/Csrf.php';

$session = [];
$token = Csrf::token($session);
echo "Token issued, length=" . strlen($token) . "\n";

var_dump(Csrf::verify($session, $token));            // matches the session
var_dump(Csrf::verify($session, 'forged-token'));     // an attacker's guess
```

```text
Token issued, length=64
bool(true)
bool(false)
```

`bin2hex(random_bytes(32))` is what makes the token unguessable —
`random_bytes()` uses a cryptographically secure random source, unlike
`rand()` or `mt_rand()`, which are predictable enough to be attacked. In a
real form, the token is rendered as a hidden input
(`<input type="hidden" name="csrf_token" value="...">`) and `verify()` is
called against the submitted `$_POST['csrf_token']` before the request is
allowed to do anything.

## Password hashing at scale: same primitive, more care

```php
<?php
declare(strict_types=1);

$hash = password_hash('correct horse battery staple', PASSWORD_BCRYPT);
echo "Hash starts with: " . substr($hash, 0, 7) . "\n";

var_dump(password_verify('correct horse battery staple', $hash));  // true
var_dump(password_verify('wrong password', $hash));                // false
```

```text
Hash starts with: $2y$12$
bool(true)
bool(false)
```

`password_hash()` was already covered in Level 3; at scale the additional
concern is `password_needs_rehash()` — checking, on every successful login,
whether the *cost parameter* used to create a stored hash has fallen behind
your current default. Hardware gets faster, so a cost that was strong in
2020 is weaker (relatively) years later; `password_needs_rehash($hash,
PASSWORD_BCRYPT)` returns `true` when it's worth re-hashing with today's
settings, letting a fleet of users' hashes gradually strengthen as they log
in, without a disruptive mass migration.

## PHP traps

**`===` string comparison for tokens leaks timing information.**
`$expected === $submitted` short-circuits at the first differing byte,
making the comparison *microseconds* faster for a wrong guess that differs
early versus one that differs late — enough of a signal, over thousands of
requests, for a timing attack to reconstruct a secret byte by byte.
`hash_equals()` always takes the same amount of time regardless of where
the strings diverge — use it for **any** secret comparison (CSRF tokens,
API keys, HMAC signatures), never a plain `===`.

**`.env` in `.gitignore` doesn't protect a file that's already committed.**
Adding a secrets file to `.gitignore` *after* it was committed once does
nothing — the value is still in git history, retrievable by anyone with
clone access, forever, until the repository history itself is rewritten
(and the leaked secret rotated regardless, since a rewrite doesn't undo
whoever already cloned it).

**A CSRF token stored in session but validated across a different session
scope silently disables the protection.** If session cookies aren't
scoped correctly (e.g. missing `SameSite` attribute, or a session that
resets on every request due to a cookie misconfiguration), `verify()` can
end up comparing against an empty or freshly regenerated token and *always*
returning false — which developers "fix" by disabling CSRF checking rather
than fixing the session bug, silently reopening the hole.

## Security-at-scale cheat sheet

| Concern | Tool | Key rule |
|---|---|---|
| Secrets | `.env` / platform secret store | Never commit; access through one central reader |
| Missing secret | `require()`-style loud failure | Fail at startup, not deep in a request |
| CSRF | Random token per session, checked per state-changing request | `hash_equals()`, never `===` |
| Token/secret comparison | `hash_equals($expected, $actual)` | Constant-time, avoids timing attacks |
| Password storage | `password_hash()` / `password_verify()` | Never store or log plaintext |
| Aging hashes | `password_needs_rehash()` | Check and upgrade on successful login |

## Exercise

Extend `EnvSecrets` with a `mask(string $key): string` method that returns
the secret's value with all but the last 4 characters replaced by `*` (e.g.
`s3cret` -> `**cret`), useful for logging "which API key is configured"
without leaking it. Then write a small `verifyLogin(string $submittedToken,
array &$session, string $password, string $storedHash): bool` function that
returns `false` immediately if the CSRF token fails `Csrf::verify()` (never
even checking the password in that case), and otherwise returns the result
of `password_verify()`. Test it with a forged CSRF token (expect `false`
without touching the password), a valid token with the wrong password
(expect `false`), and a valid token with the right password (expect `true`).
