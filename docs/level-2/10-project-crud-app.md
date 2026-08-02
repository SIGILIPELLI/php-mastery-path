# 10 · Project — CRUD Web App with SQLite

A capstone project combining everything from Level 2: [OOP](01-oop-deep-dive.md)
(interfaces stay implicit here, but classes, encapsulation, and validation
are central), [PDO with SQLite](03-databases-pdo.md), [sessions, forms, CSRF
protection, and login](04-sessions-forms.md), and [PHPUnit tests](05-phpunit-testing.md)
for the data layer.

## What you'll build

A small logged-in "Notes" app:

- Log in with a demo username/password (session-based, CSRF-protected)
- List all notes
- Create a note (server-side validated)
- Edit a note
- Delete a note
- Everything persisted to a SQLite file via PDO — no separate database
  server needed
- A PHPUnit test suite covering the data layer (`NoteRepository`)

## Project layout

```text
notes_app/
    composer.json
    src/
        Database.php
        Note.php
        NoteRepository.php
        Auth.php
    public/
        _bootstrap.php
        login.php
        logout.php
        index.php
        create.php
        edit.php
        delete.php
    tests/
        NoteRepositoryTest.php
```

## `composer.json`

```json
{
    "name": "demo/notes-app",
    "require": {
        "php": ">=8.2",
        "ext-pdo": "*",
        "ext-pdo_sqlite": "*"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    }
}
```

Run `composer install` once in `notes_app/` to generate `vendor/autoload.php`
and download PHPUnit.

## `src/Database.php` — connection + schema

```php
<?php
declare(strict_types=1);

namespace App;

use PDO;

class Database
{
    // Defaults to a real file so data survives between requests; tests
    // pass ":memory:" instead for a fresh, throwaway database per run.
    public static function connect(string $path = __DIR__ . "/../data/notes.sqlite"): PDO
    {
        if ($path !== ":memory:" && !is_dir(dirname($path))) {
            mkdir(dirname($path), recursive: true);
        }

        $pdo = new PDO("sqlite:" . $path);
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

        $pdo->exec("
            CREATE TABLE IF NOT EXISTS notes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                body TEXT NOT NULL,
                created_at TEXT NOT NULL
            )
        ");

        return $pdo;
    }
}
```

## `src/Note.php` — the domain object

```php
<?php
declare(strict_types=1);

namespace App;

use InvalidArgumentException;

class Note
{
    public function __construct(
        public readonly ?int $id,
        public string $title,
        public string $body,
        public readonly string $createdAt = "",
    ) {
        $this->validate();
    }

    private function validate(): void
    {
        if (trim($this->title) === "") {
            throw new InvalidArgumentException("Title cannot be empty");
        }
        if (mb_strlen($this->title) > 120) {
            throw new InvalidArgumentException("Title must be 120 characters or fewer");
        }
    }
}
```

Validation lives inside the constructor, not in the web form handler — so
`new Note(...)` can **never** produce an invalid object, no matter which
code path creates one (a form, a test, a future import script).

## `src/NoteRepository.php` — the PDO data layer

```php
<?php
declare(strict_types=1);

namespace App;

use PDO;

class NoteRepository
{
    public function __construct(private PDO $pdo) {}

    /** @return Note[] */
    public function all(): array
    {
        $rows = $this->pdo
            ->query("SELECT * FROM notes ORDER BY id DESC")
            ->fetchAll(PDO::FETCH_ASSOC);

        return array_map($this->hydrate(...), $rows);
    }

    public function find(int $id): ?Note
    {
        $stmt = $this->pdo->prepare("SELECT * FROM notes WHERE id = :id");
        $stmt->execute(["id" => $id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        return $row === false ? null : $this->hydrate($row);
    }

    public function create(Note $note): int
    {
        $stmt = $this->pdo->prepare(
            "INSERT INTO notes (title, body, created_at) VALUES (:title, :body, :created_at)"
        );
        $stmt->execute([
            "title" => $note->title,
            "body" => $note->body,
            "created_at" => date(DATE_ATOM),
        ]);

        return (int) $this->pdo->lastInsertId();
    }

    public function update(int $id, Note $note): void
    {
        $stmt = $this->pdo->prepare(
            "UPDATE notes SET title = :title, body = :body WHERE id = :id"
        );
        $stmt->execute(["title" => $note->title, "body" => $note->body, "id" => $id]);
    }

    public function delete(int $id): void
    {
        $stmt = $this->pdo->prepare("DELETE FROM notes WHERE id = :id");
        $stmt->execute(["id" => $id]);
    }

    private function hydrate(array $row): Note
    {
        return new Note((int) $row["id"], $row["title"], $row["body"], $row["created_at"]);
    }
}
```

Every query that includes a variable uses a prepared statement with named
placeholders — never string concatenation — so note titles and bodies can
contain any characters (quotes, SQL keywords, anything) with zero risk of
SQL injection.

## `src/Auth.php` — session-based login

```php
<?php
declare(strict_types=1);

namespace App;

class Auth
{
    // Hard-coded single user for a demo -- a real app looks credentials up
    // in a database table, one password_hash() per user, never in source.
    private const USERNAME = "admin";
    private const PASSWORD = "secret123";

    public static function attempt(string $username, string $password): bool
    {
        $validHash = password_hash(self::PASSWORD, PASSWORD_DEFAULT);

        if ($username === self::USERNAME && password_verify($password, $validHash)) {
            session_regenerate_id(delete_old_session: true);   // defeat session fixation
            $_SESSION["user"] = $username;
            return true;
        }
        return false;
    }

    public static function check(): bool
    {
        return isset($_SESSION["user"]);
    }

    public static function requireLogin(): void
    {
        if (!self::check()) {
            header("Location: login.php");
            exit;
        }
    }

    public static function logout(): void
    {
        $_SESSION = [];
        session_destroy();
    }
}
```

## `public/_bootstrap.php` — shared setup for every page

```php
<?php
declare(strict_types=1);

require __DIR__ . "/../vendor/autoload.php";

session_start();
$_SESSION["csrf_token"] ??= bin2hex(random_bytes(32));
```

## `public/login.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . "/_bootstrap.php";

use App\Auth;

$error = null;

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    if (!hash_equals($_SESSION["csrf_token"], $_POST["csrf_token"] ?? "")) {
        $error = "Invalid form submission, please try again.";
    } elseif (Auth::attempt(trim($_POST["username"] ?? ""), $_POST["password"] ?? "")) {
        header("Location: index.php");
        exit;
    } else {
        $error = "Invalid username or password.";
    }
}
?>
<!DOCTYPE html>
<html>
<head><title>Log in — Notes</title></head>
<body>
    <h1>Log in</h1>
    <?php if ($error !== null): ?>
        <p style="color:red;"><?= htmlspecialchars($error) ?></p>
    <?php endif; ?>
    <form method="post" action="login.php">
        <input type="hidden" name="csrf_token" value="<?= htmlspecialchars($_SESSION["csrf_token"]) ?>">
        <label>Username <input type="text" name="username"></label><br>
        <label>Password <input type="password" name="password"></label><br>
        <button type="submit">Log in</button>
    </form>
    <p><em>Demo credentials: admin / secret123</em></p>
</body>
</html>
```

## `public/index.php` — list notes

```php
<?php
declare(strict_types=1);

require __DIR__ . "/_bootstrap.php";

use App\Auth;
use App\Database;
use App\NoteRepository;

Auth::requireLogin();

$notes = (new NoteRepository(Database::connect()))->all();
?>
<!DOCTYPE html>
<html>
<head><title>My Notes</title></head>
<body>
    <h1>My Notes</h1>
    <p>
        Logged in as <?= htmlspecialchars($_SESSION["user"]) ?>
        — <a href="logout.php">Log out</a>
    </p>
    <p><a href="create.php">+ New note</a></p>
    <ul>
        <?php foreach ($notes as $note): ?>
            <li>
                <strong><?= htmlspecialchars($note->title) ?></strong>
                <small>(<?= htmlspecialchars($note->createdAt) ?>)</small>
                — <a href="edit.php?id=<?= $note->id ?>">Edit</a>
                — <form method="post" action="delete.php" style="display:inline">
                    <input type="hidden" name="id" value="<?= $note->id ?>">
                    <input type="hidden" name="csrf_token" value="<?= htmlspecialchars($_SESSION["csrf_token"]) ?>">
                    <button type="submit" onclick="return confirm('Delete this note?')">Delete</button>
                </form>
            </li>
        <?php endforeach; ?>
        <?php if (empty($notes)): ?>
            <li><em>No notes yet.</em></li>
        <?php endif; ?>
    </ul>
</body>
</html>
```

## `public/create.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . "/_bootstrap.php";

use App\Auth;
use App\Database;
use App\Note;
use App\NoteRepository;

Auth::requireLogin();

$error = null;

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    if (!hash_equals($_SESSION["csrf_token"], $_POST["csrf_token"] ?? "")) {
        $error = "Invalid form submission, please try again.";
    } else {
        try {
            $note = new Note(null, trim($_POST["title"] ?? ""), trim($_POST["body"] ?? ""));
            (new NoteRepository(Database::connect()))->create($note);
            header("Location: index.php");
            exit;
        } catch (InvalidArgumentException $e) {
            $error = $e->getMessage();
        }
    }
}
?>
<!DOCTYPE html>
<html>
<head><title>New Note</title></head>
<body>
    <h1>New Note</h1>
    <?php if ($error !== null): ?>
        <p style="color:red;"><?= htmlspecialchars($error) ?></p>
    <?php endif; ?>
    <form method="post" action="create.php">
        <input type="hidden" name="csrf_token" value="<?= htmlspecialchars($_SESSION["csrf_token"]) ?>">
        <label>Title <input type="text" name="title" value="<?= htmlspecialchars($_POST["title"] ?? "") ?>"></label><br>
        <label>Body <textarea name="body"><?= htmlspecialchars($_POST["body"] ?? "") ?></textarea></label><br>
        <button type="submit">Save</button>
    </form>
    <p><a href="index.php">Back to list</a></p>
</body>
</html>
```

## `public/edit.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . "/_bootstrap.php";

use App\Auth;
use App\Database;
use App\Note;
use App\NoteRepository;

Auth::requireLogin();

$repository = new NoteRepository(Database::connect());
$id = (int) ($_GET["id"] ?? $_POST["id"] ?? 0);
$note = $repository->find($id);

if ($note === null) {
    http_response_code(404);
    exit("Note not found.");
}

$error = null;

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    if (!hash_equals($_SESSION["csrf_token"], $_POST["csrf_token"] ?? "")) {
        $error = "Invalid form submission, please try again.";
    } else {
        try {
            $updated = new Note($id, trim($_POST["title"] ?? ""), trim($_POST["body"] ?? ""));
            $repository->update($id, $updated);
            header("Location: index.php");
            exit;
        } catch (InvalidArgumentException $e) {
            $error = $e->getMessage();
        }
    }
}
?>
<!DOCTYPE html>
<html>
<head><title>Edit Note</title></head>
<body>
    <h1>Edit Note</h1>
    <?php if ($error !== null): ?>
        <p style="color:red;"><?= htmlspecialchars($error) ?></p>
    <?php endif; ?>
    <form method="post" action="edit.php">
        <input type="hidden" name="id" value="<?= $note->id ?>">
        <input type="hidden" name="csrf_token" value="<?= htmlspecialchars($_SESSION["csrf_token"]) ?>">
        <label>Title <input type="text" name="title" value="<?= htmlspecialchars($note->title) ?>"></label><br>
        <label>Body <textarea name="body"><?= htmlspecialchars($note->body) ?></textarea></label><br>
        <button type="submit">Update</button>
    </form>
    <p><a href="index.php">Back to list</a></p>
</body>
</html>
```

## `public/delete.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . "/_bootstrap.php";

use App\Auth;
use App\Database;
use App\NoteRepository;

Auth::requireLogin();

if ($_SERVER["REQUEST_METHOD"] === "POST"
    && hash_equals($_SESSION["csrf_token"], $_POST["csrf_token"] ?? "")
) {
    $id = (int) ($_POST["id"] ?? 0);
    (new NoteRepository(Database::connect()))->delete($id);
}

header("Location: index.php");
exit;
```

## `public/logout.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . "/_bootstrap.php";

use App\Auth;

Auth::logout();
header("Location: login.php");
exit;
```

## `tests/NoteRepositoryTest.php` — testing the data layer

The web pages glue HTTP, sessions, and the data layer together — but the
*logic* worth testing automatically is `NoteRepository` and `Note`, which
have no dependency on `$_POST`, `$_SESSION`, or a running web server. Each
test gets its own throwaway `:memory:` database, so tests never interfere
with each other or with the real `data/notes.sqlite` file.

```php
<?php
declare(strict_types=1);

namespace Tests;

use App\Database;
use App\Note;
use App\NoteRepository;
use InvalidArgumentException;
use PHPUnit\Framework\TestCase;

class NoteRepositoryTest extends TestCase
{
    private NoteRepository $repository;

    protected function setUp(): void
    {
        $this->repository = new NoteRepository(Database::connect(":memory:"));
    }

    public function testCreateAndFind(): void
    {
        $id = $this->repository->create(new Note(null, "Groceries", "Milk, eggs, bread"));

        $note = $this->repository->find($id);

        $this->assertNotNull($note);
        $this->assertSame("Groceries", $note->title);
        $this->assertSame("Milk, eggs, bread", $note->body);
    }

    public function testFindReturnsNullForMissingId(): void
    {
        $this->assertNull($this->repository->find(999));
    }

    public function testAllReturnsNewestFirst(): void
    {
        $this->repository->create(new Note(null, "First", "..."));
        $this->repository->create(new Note(null, "Second", "..."));

        $notes = $this->repository->all();

        $this->assertCount(2, $notes);
        $this->assertSame("Second", $notes[0]->title);   // most recently created, listed first
    }

    public function testUpdateChangesStoredFields(): void
    {
        $id = $this->repository->create(new Note(null, "Old title", "Old body"));

        $this->repository->update($id, new Note($id, "New title", "New body"));

        $updated = $this->repository->find($id);
        $this->assertSame("New title", $updated->title);
        $this->assertSame("New body", $updated->body);
    }

    public function testDeleteRemovesTheNote(): void
    {
        $id = $this->repository->create(new Note(null, "Temporary", "..."));

        $this->repository->delete($id);

        $this->assertNull($this->repository->find($id));
    }

    public function testEmptyTitleIsRejected(): void
    {
        $this->expectException(InvalidArgumentException::class);

        new Note(null, "   ", "A body with no real title");
    }
}
```

## Running it

```bash
cd notes_app
composer install

# Serve the app with PHP's built-in development server
php -S localhost:8000 -t public

# Visit http://localhost:8000/login.php in a browser, log in with
# admin / secret123, and create, edit, and delete a few notes.

# In a separate terminal, run the automated test suite:
vendor/bin/phpunit tests
```

Expected test output:

```text
PHPUnit 11.x by Sebastian Bergmann and contributors.

......                                                              6 / 6 (100%)

OK (6 tests, 10 assertions)
```

## Stretch goals

- Add a `tags` column (comma-separated or a proper join table) and a filter
  form on `index.php`.
- Add pagination to `index.php` once note counts grow — `LIMIT`/`OFFSET` in
  `NoteRepository::all()`.
- Move the hard-coded `Auth` credentials into a `users` table, hashing each
  password with `password_hash()` at account-creation time instead of at
  login time.
- Add an `updated_at` column and display "last edited X" using the
  [`DateTime` tools](09-working-with-dates.md).
- Write a PHPUnit test for `Auth::attempt()` — note that it touches
  `$_SESSION`, so you'll need to start a session (or use PHPUnit's
  `@runInSeparateProcess` annotation) for it to behave predictably in tests.
