# 10 · Project — CLI To-Do App

A small end-to-end project combining everything from Level 1: functions,
arrays, strings, classes, error handling, and Composer basics.

## What you'll build

A command-line to-do list that:

- Adds tasks
- Lists tasks (with done/pending status)
- Marks tasks done
- Deletes tasks
- Persists everything to a JSON file between runs

## Project layout

```text
todo_app/
    TaskStorage.php
    todo.php
```

## TaskStorage.php — persistence layer

```php
<?php
// TaskStorage.php
declare(strict_types=1);

class TaskStorage {
    private string $path;

    public function __construct(string $path = "tasks.json") {
        $this->path = $path;
    }

    public function load(): array {
        if (!file_exists($this->path)) {
            return [];
        }

        $contents = file_get_contents($this->path);
        $tasks = json_decode($contents, true);

        return is_array($tasks) ? $tasks : [];
    }

    public function save(array $tasks): void {
        file_put_contents($this->path, json_encode($tasks, JSON_PRETTY_PRINT));
    }
}
```

## todo.php — CLI logic

```php
<?php
// todo.php
declare(strict_types=1);

require_once __DIR__ . "/TaskStorage.php";

function addTask(array &$tasks, string $description): void {
    $tasks[] = ["description" => $description, "done" => false];
    echo "Added: $description\n";
}

function listTasks(array $tasks): void {
    if (empty($tasks)) {
        echo "No tasks yet.\n";
        return;
    }

    foreach ($tasks as $i => $task) {
        $status = $task["done"] ? "x" : " ";
        $num = $i + 1;
        echo "[$status] $num. {$task['description']}\n";
    }
}

function completeTask(array &$tasks, int $index): void {
    if (!isset($tasks[$index - 1])) {
        echo "No task with number $index\n";
        return;
    }
    $tasks[$index - 1]["done"] = true;
    echo "Marked task $index done.\n";
}

function deleteTask(array &$tasks, int $index): void {
    if (!isset($tasks[$index - 1])) {
        echo "No task with number $index\n";
        return;
    }
    $removed = array_splice($tasks, $index - 1, 1)[0];
    echo "Deleted: {$removed['description']}\n";
}

$storage = new TaskStorage();
$tasks = $storage->load();

$args = array_slice($argv, 1);   // $argv[0] is the script name itself

if (empty($args)) {
    echo "Usage: php todo.php [add <text> | list | done <n> | delete <n>]\n";
    exit(0);
}

$command = $args[0];

switch ($command) {
    case "add":
        if (count($args) < 2) {
            echo "Usage: php todo.php add <text>\n";
            break;
        }
        addTask($tasks, implode(" ", array_slice($args, 1)));
        $storage->save($tasks);
        break;

    case "list":
        listTasks($tasks);
        break;

    case "done":
        completeTask($tasks, (int) ($args[1] ?? 0));
        $storage->save($tasks);
        break;

    case "delete":
        deleteTask($tasks, (int) ($args[1] ?? 0));
        $storage->save($tasks);
        break;

    default:
        echo "Unknown command: $command\n";
}
```

## Running it

```bash
php todo.php add "Write Level 1 exercises"
php todo.php add "Review Level 2 outline"
php todo.php list
# [ ] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline

php todo.php done 1
php todo.php list
# [x] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline
```

## Stretch goals

- Add a `priority` field (`low`/`medium`/`high`) and sort the list by it.
- Add due dates using PHP's `DateTime` class, and highlight overdue tasks.
- Add a basic PHPUnit test for `TaskStorage` (you'll formalize this properly
  with PHPUnit in [Level 2](../level-2/05-phpunit-testing.md)).

Completing this project means you're ready for **Level 2 · Intermediate**.
