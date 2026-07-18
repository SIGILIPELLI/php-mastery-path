# 01 · Setup & First Program

PHP is a server-side scripting language: your script runs on a server (or your
own machine, while learning), and produces output — usually HTML, JSON, or
plain text. Unlike Java, there's no separate compile step you run by hand —
the `php` interpreter reads and executes your source file directly.

## Installing PHP

Install PHP 8.x. The examples in this whole training program assume PHP 8.1+.

```bash
# macOS (Homebrew)
brew install php

# Ubuntu/Debian
sudo apt update
sudo apt install php-cli

# Windows: download from https://windows.php.net/download and add the
# extracted folder to your PATH, or install via https://laragon.org/ which
# bundles PHP, a web server, and MySQL together.
```

Verify the install:

```bash
php --version
# PHP 8.3.x (cli) (built: ...) ...
```

`php` is the CLI (command-line interface) binary — it can run scripts
directly, start a lightweight dev server, or drop you into an interactive
shell with `php -a`.

## Your first script

Create a file named `hello.php`:

```php
<?php
// Every PHP file that contains PHP code starts with this opening tag.
// Code between "<?php" and the (optional) closing "?>" is executed;
// anything outside it is passed through as plain text/HTML.

echo "Hello, world!\n";
```

Run it from the command line:

```bash
php hello.php
# Hello, world!
```

`echo` is a language construct (not a function) that outputs one or more
strings. `\n` inside a double-quoted string is an escape sequence for a
newline — single-quoted strings do **not** interpret escape sequences (more
on that in [Module 2](02-variables-types.md)).

## Anatomy of a PHP file

| Piece | Meaning |
|-------|---------|
| `<?php` | Opening tag — switches the parser from "HTML/text mode" into "PHP mode." |
| `?>` | Closing tag — optional at the end of a pure-PHP file, and usually **omitted** to avoid accidental trailing whitespace/output. |
| `echo`, `print` | Output constructs — `echo` can take multiple comma-separated arguments, `print` always returns `1` and takes exactly one. |
| `;` | Every statement ends with a semicolon. |
| `//`, `#` | Single-line comments. |
| `/* ... */` | Multi-line comment. |

Because PHP was designed to be embedded inside HTML, you can mix modes freely
— this is common in older PHP and in templates, less common in modern
application code that separates logic from rendering:

```php
<!DOCTYPE html>
<html>
<body>
    <h1>
        <?php
        // Anything inside <?php ... ?> is executed; the rest is literal HTML.
        echo "Welcome!";
        ?>
    </h1>
    <p>Today is <?= date("Y-m-d") ?></p>
    <!-- "<?=" is shorthand for "<?php echo" -->
</body>
</html>
```

## Running a script vs. serving a site

`php hello.php` runs a script once from the command line and prints its
output to your terminal — great for learning the language and for CLI tools.
Real web apps need a server that listens for HTTP requests and runs your PHP
per-request. PHP ships a **built-in development server** for exactly this,
so you don't need to install Apache or Nginx while learning:

```bash
# Serve the current directory at http://localhost:8000
php -S localhost:8000
```

Create an `index.php`:

```php
<?php
// This runs fresh on every request the built-in server receives.
echo "Served by PHP's built-in dev server!\n";
echo "You requested: " . $_SERVER['REQUEST_URI'];
```

With `php -S localhost:8000` running in that directory, visiting
`http://localhost:8000/` in a browser (or `curl http://localhost:8000/`)
executes `index.php` and returns its output. Stop the server with `Ctrl+C`.
The built-in server is single-threaded and meant for development only — never
use it in production.

## Choosing an editor

**VS Code** with the "PHP Intelephense" extension is the most common free
setup — good autocompletion and inline error-checking without a heavy IDE.
**PhpStorm** (paid, free for students) is the deepest PHP-specific IDE if you
want refactoring tools and a built-in debugger. Either is fine for this
course — the language matters far more than the editor.

## A note on Composer

Real-world PHP projects almost always pull in libraries via **Composer**,
PHP's dependency manager (similar to `npm` for Node or `pip` for Python).
Everything through [Module 8](08-error-handling.md) only uses plain PHP with
no dependencies, so you don't need it yet — Composer gets its own full
treatment in [Module 9 · Composer & Package Basics](09-composer-basics.md).

## Exercise

Create a file `greet.php` that:

1. Uses `echo` to print a greeting for three different names, one per line.
2. Uses the `<?= ?>` short-echo tag at least once.

Run it with `php greet.php` from your terminal. Then start
`php -S localhost:8000` in the same folder, rename the file to `index.php`,
and confirm you see the same output by visiting `http://localhost:8000/` in a
browser.
