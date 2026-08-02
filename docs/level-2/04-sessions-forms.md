# 04 · Sessions & Forms

Web apps are stateless by default — every HTTP request arrives with no
memory of the last one. **Sessions** give a visitor a small pocket of
server-side storage that persists across requests (so "logged in" can mean
something), and **forms** are how a visitor sends data back to your script
in the first place. Both come with well-known security traps that are worth
understanding from day one.

## Reading form input: `$_GET` vs `$_POST`

```php
<?php
// search.php -- reached via a GET form: <form method="get" action="search.php">
if ($_SERVER["REQUEST_METHOD"] === "GET" && isset($_GET["q"])) {
    $query = trim($_GET["q"]);
    echo "You searched for: " . htmlspecialchars($query);
}
```

```html
<!-- login.php -- reached via a POST form -->
<form method="post" action="login.php">
    <input type="text" name="username">
    <input type="password" name="password">
    <button type="submit">Log in</button>
</form>
```

```php
<?php
// login.php
if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $username = trim($_POST["username"] ?? "");
    $password = $_POST["password"] ?? "";
    // ... validate and log in below
}
```

`$_GET` values come from the URL's query string (visible in browser history,
bookmarkable, size-limited) — use it for searches and filters, never for
passwords or anything sensitive. `$_POST` values come from the request body
and are the right choice for anything that changes state (logging in,
submitting a purchase, deleting a record) or contains sensitive data.

## Always escape output: preventing XSS

If a form value is ever echoed back into HTML without escaping, a visitor
who submits `<script>...</script>` as their input can run arbitrary
JavaScript in every other visitor's browser who sees that output — a
**cross-site scripting (XSS)** attack. `htmlspecialchars()` converts HTML's
special characters into harmless text.

```php
<?php

$comment = '<script>alert("hacked")</script>';

echo $comment;                       // DANGEROUS: browser executes the script
echo htmlspecialchars($comment);     // SAFE: prints the literal text
// &lt;script&gt;alert(&quot;hacked&quot;)&lt;/script&gt;
```

The rule of thumb: escape on **output**, every time you print anything that
originated from user input, not just once "when it comes in."

## Validating form input

```php
<?php

function validateSignup(array $input): array
{
    $errors = [];

    $email = trim($input["email"] ?? "");
    if ($email === "" || !filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $errors["email"] = "Enter a valid email address";
    }

    $age = $input["age"] ?? "";
    if (!ctype_digit((string) $age) || (int) $age < 13) {
        $errors["age"] = "You must be at least 13 years old";
    }

    $password = $input["password"] ?? "";
    if (strlen($password) < 8) {
        $errors["password"] = "Password must be at least 8 characters";
    }

    return $errors;
}

$submission = ["email" => "not-an-email", "age" => "9", "password" => "short"];
$errors = validateSignup($submission);

foreach ($errors as $field => $message) {
    echo "$field: $message\n";
}
// email: Enter a valid email address
// age: You must be at least 13 years old
// password: Password must be at least 8 characters
```

Validate every field server-side even if you also validate in JavaScript —
client-side checks are a convenience for honest users, not a security
boundary; anyone can submit a request directly and skip your JavaScript
entirely.

## Starting a session and storing data

```php
<?php
// Must run before ANY output (even a stray blank line) -- it sends a
// Set-Cookie header, and headers can't be sent after the response body starts.
session_start();

$_SESSION["cart"] ??= [];             // initialize once
$_SESSION["cart"][] = "Widget";       // persists across requests for this visitor

echo "Items in cart: " . count($_SESSION["cart"]) . "\n";
```

A session works by giving the browser a cookie (`PHPSESSID` by default)
containing a random ID; PHP uses that ID to load a matching file of
serialized data on the server for every subsequent request from that
browser. The visitor never sees the actual stored data, only the ID.

## Logging in safely: session fixation

**Session fixation** is an attack where someone tricks a victim into using a
session ID the attacker already knows (e.g. via a crafted link), then waits
for the victim to log in under that same ID — hijacking the now-authenticated
session. The fix is simple: always issue a brand-new session ID the moment
privilege changes (typically, right after a successful login).

```php
<?php

session_start();

function attemptLogin(string $username, string $password): bool
{
    $validUsername = "ada";
    $validPasswordHash = password_hash("s3cret!", PASSWORD_DEFAULT);

    if ($username === $validUsername && password_verify($password, $validPasswordHash)) {
        // Issue a fresh session ID -- invalidates any ID an attacker
        // might have fixed beforehand, and deletes the old session file.
        session_regenerate_id(delete_old_session: true);
        $_SESSION["user"] = $username;
        return true;
    }
    return false;
}

if (attemptLogin("ada", "s3cret!")) {
    echo "Logged in as: " . $_SESSION["user"] . "\n";
    // Logged in as: ada
}
```

Never store or compare plaintext passwords — `password_hash()` (one-way,
salted) and `password_verify()` are the standard library functions for this;
never roll your own hashing scheme.

## CSRF: making sure the request really came from your form

**Cross-site request forgery (CSRF)** tricks a logged-in visitor's browser
into submitting a request to your site from a *different* site — the
browser automatically attaches the victim's session cookie, so the request
looks legitimate. The defense is a random token embedded in your own form
that an attacker's page can't guess or read.

```php
<?php
session_start();

// When rendering the form:
$_SESSION["csrf_token"] ??= bin2hex(random_bytes(32));
$token = $_SESSION["csrf_token"];
echo '<input type="hidden" name="csrf_token" value="' . htmlspecialchars($token) . '">';

// When handling the POST:
$submitted = $_POST["csrf_token"] ?? "";
if (!hash_equals($_SESSION["csrf_token"] ?? "", $submitted)) {
    http_response_code(403);
    exit("Invalid or missing CSRF token.");
}
```

`hash_equals()` compares strings in constant time, avoiding a timing attack
that could otherwise let an attacker guess the token one character at a
time by measuring how long comparisons take.

## Sessions & forms cheat sheet

| Tool | Purpose |
|------|---------|
| `$_GET`, `$_POST` | Read query-string vs. request-body input |
| `htmlspecialchars($value)` | Escape output to prevent XSS |
| `filter_var($v, FILTER_VALIDATE_EMAIL)` | Validate common formats |
| `session_start()` | Begin/resume a session (call before any output) |
| `$_SESSION` | Server-side, per-visitor storage across requests |
| `session_regenerate_id()` | Issue a fresh session ID (call on login) |
| `password_hash()` / `password_verify()` | Safely store and check passwords |
| `hash_equals()` | Timing-safe string comparison, e.g. for CSRF tokens |

## Exercise

Build a two-file login demo: `login_form.php` renders a form with username,
password, and a hidden CSRF token stored in `$_SESSION`; `login.php` handles
the `POST`, rejects the request if the CSRF token doesn't match
(`hash_equals`), validates the username/password against a hard-coded
`password_hash()`ed value, and on success calls `session_regenerate_id()`
before storing `$_SESSION["user"]`. Add a `logout.php` that calls
`session_destroy()`. Test the flow with PHP's built-in server:
`php -S localhost:8000`.
