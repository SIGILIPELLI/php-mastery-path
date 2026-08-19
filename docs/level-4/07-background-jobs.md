# 07 · Background Job Processing

[Working with Queues](../level-3/06-working-with-queues.md) built the
transport — pushing raw job data onto a queue and popping it off in a
worker. This module covers the layer on top: modeling **jobs as classes**
with their own `handle()` logic (instead of raw arrays a worker has to
`switch` on), dispatching them from application code, and running
**scheduled** (cron-style) tasks — the other common shape background work
takes.

## Jobs as classes, not raw payloads

A queue that stores `['type' => 'send_email', 'to' => '...']` and a worker
`switch`-ing on `type` works, but grows unwieldy fast — every new job type
adds a `case` and the actual logic ends up scattered wherever the switch
lives. Modeling each job as a class implementing a common interface keeps
each job's logic (and its data) together, and adding a new job type never
means touching existing ones.

```php
<?php
// Job.php
declare(strict_types=1);

interface Job
{
    public function handle(): void;
    public function payload(): array;   // for serializing onto a real queue transport
}
```

```php
<?php
// SendWelcomeEmailJob.php, GenerateReportJob.php
declare(strict_types=1);

final class SendWelcomeEmailJob implements Job
{
    public function __construct(private string $email) {}

    public function handle(): void
    {
        echo "Sending welcome email to {$this->email}\n";
    }

    public function payload(): array
    {
        return ['type' => self::class, 'email' => $this->email];
    }
}

final class GenerateReportJob implements Job
{
    public function __construct(private int $reportId) {}

    public function handle(): void
    {
        echo "Generating report #{$this->reportId}\n";
    }

    public function payload(): array
    {
        return ['type' => self::class, 'reportId' => $this->reportId];
    }
}
```

## A dispatcher: the application-facing API

Application code shouldn't know or care whether a job runs immediately or
gets queued for a worker — `JobDispatcher::dispatch()` is the one place
that decision lives.

```php
<?php
// JobDispatcher.php
declare(strict_types=1);

final class JobDispatcher
{
    /** @var Job[] */
    private array $queue = [];

    public function dispatch(Job $job): void
    {
        $this->queue[] = $job;
        echo "Dispatched: " . get_class($job) . "\n";
    }

    /** Bypass the queue entirely -- useful for CLI commands and tests. */
    public function runSync(Job $job): void
    {
        $job->handle();
    }

    /** What a worker process calls in a loop. */
    public function processQueue(): int
    {
        $count = 0;
        while ($job = array_shift($this->queue)) {
            $job->handle();
            $count++;
        }
        return $count;
    }
}
```

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/Job.php';
require __DIR__ . '/SendWelcomeEmailJob.php';
require __DIR__ . '/GenerateReportJob.php';
require __DIR__ . '/JobDispatcher.php';

$dispatcher = new JobDispatcher();
$dispatcher->dispatch(new SendWelcomeEmailJob('ada@example.com'));
$dispatcher->dispatch(new GenerateReportJob(42));

echo "Queue built, now processing on a worker...\n";
$processed = $dispatcher->processQueue();
echo "Processed $processed jobs\n";
```

```text
Dispatched: SendWelcomeEmailJob
Dispatched: GenerateReportJob
Queue built, now processing on a worker...
Sending welcome email to ada@example.com
Generating report #42
Processed 2 jobs
```

In this in-memory demo, `dispatch()` and `processQueue()` run in the same
process for clarity. In a real system, `dispatch()` would call
`payload()` and hand the result to a real transport — the `FileQueue` from
[Working with Queues](../level-3/06-working-with-queues.md), or Redis/RabbitMQ
in production — and a *separate, long-running worker process* would pop
payloads, look up the class by `payload['type']`, reconstruct the job, and
call `handle()`. The dispatcher's interface stays identical either way;
only what's behind `dispatch()` changes.

## Scheduled (cron-style) tasks

Not all background work is triggered by an event (a user signing up). Some
runs on a fixed interval — clearing expired sessions every hour, sending a
daily digest. The underlying mechanism is a **cron entry** invoking a PHP
script on a schedule; inside that script, a simple "is this due yet" check
lets multiple scheduled tasks share one process.

```php
<?php
// ScheduledTask.php
declare(strict_types=1);

final class ScheduledTask
{
    public function __construct(
        private string $name,
        private int $intervalSeconds,
        private int $lastRun = 0,
    ) {}

    public function isDue(int $now): bool
    {
        return ($now - $this->lastRun) >= $this->intervalSeconds;
    }

    public function run(int $now): void
    {
        echo "Running scheduled task '{$this->name}' at " . date('H:i:s', $now) . "\n";
        $this->lastRun = $now;
    }
}
```

```php
<?php
// demo.php (continued)
declare(strict_types=1);

$task = new ScheduledTask('cleanup-temp-files', 5); // every 5 "seconds" for this demo

$t0 = time();
var_dump($task->isDue($t0));      // never run yet -> due immediately
$task->run($t0);

var_dump($task->isDue($t0 + 2));  // only 2s since last run -> not due
var_dump($task->isDue($t0 + 6));  // 6s since last run -> due again
```

```text
bool(true)
Running scheduled task 'cleanup-temp-files' at 07:04:26
bool(false)
bool(true)
```

The real-world equivalent is a single cron entry —
`* * * * * php /path/to/artisan schedule:run` (Laravel's approach) or a
bare `* * * * * php /path/to/run-scheduler.php` — firing every minute, with
each `ScheduledTask`'s own `isDue()` (or a full cron-expression parser,
for more complex schedules) deciding whether *that particular* task
actually executes on this tick. One cron line ends up driving arbitrarily
many independently-scheduled tasks.

## PHP traps

**Reconstructing a job from `payload()` needs the class to still exist and
match its constructor.** If `SendWelcomeEmailJob`'s constructor changes
(a new required parameter) after jobs are already queued with the old
`payload()` shape, a worker deploying the new code fails trying to
reconstruct old, in-flight jobs. Real systems version job payloads or drain
the queue before deploying breaking job changes — the same discipline as
[API versioning](06-api-versioning-docs.md), applied to background work
instead of HTTP responses.

**A scheduled task with `$lastRun` stored only in memory resets every time
the process restarts.** `ScheduledTask` above is fine for a single
long-running scheduler process, but a script invoked fresh by cron every
minute needs to persist `$lastRun` somewhere durable (a database row, a
file) — otherwise every invocation thinks nothing has ever run and fires
immediately.

**`runSync()` silently hiding failures a real queue would retry.** Running
a job synchronously inside a web request (rather than dispatching it)
means any exception the job throws propagates straight into the HTTP
response — useful for tests and CLI tools, dangerous as a default, since it
reintroduces exactly the "slow/failing background work blocks the user"
problem queues exist to solve.

## Background jobs cheat sheet

| Concept | Purpose |
|---|---|
| `Job` interface | Each background task is a class: data + `handle()` logic together |
| `dispatch()` | Application-facing entry point; hides sync vs. queued behind one call |
| `runSync()` | Bypass the queue — useful for tests/CLI, risky as a request-path default |
| `payload()` | How a job serializes itself onto a real transport (Redis, `FileQueue`) |
| `ScheduledTask::isDue()` | Interval check driving cron-style recurring work |
| Single cron entry + per-task `isDue()` | One `* * * * *` line can drive many independent schedules |

## Exercise

Add a `RetryableJob` interface extending `Job` with a
`maxAttempts(): int` method, and a `FlakyReportJob implements RetryableJob`
whose `handle()` throws a `RuntimeException` on its first two calls (track
attempts in a property) and succeeds on the third, with `maxAttempts()`
returning 3. Extend `JobDispatcher::processQueue()` to catch exceptions from
jobs implementing `RetryableJob`, re-queue them (up to `maxAttempts()`),
and give up (print `"Giving up on " . get_class($job)`) once exhausted.
Dispatch one `FlakyReportJob` and confirm it eventually succeeds, printing
each attempt.
