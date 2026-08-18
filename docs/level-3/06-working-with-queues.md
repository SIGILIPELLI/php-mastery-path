# 06 · Working with Queues

Some work doesn't need to finish before you respond to the user: sending a
welcome email, resizing an uploaded image, generating a PDF report. Doing it
inline makes the request slow and makes the whole response fail if that one
step fails. A **queue** decouples "accept the work" from "do the work" — a
web request pushes a job description onto a queue and returns immediately; a
separate **worker** process pulls jobs off the queue and executes them,
independently, with retries if something goes wrong.

## A file-based queue

Production systems use a broker — Redis (via `predis/predis` or the `phpredis`
extension) or RabbitMQ (via `php-amqplib`) — because they handle
concurrency, persistence, and delivery guarantees correctly at scale. To
keep this module runnable with nothing but PHP installed, the same ideas are
built here on top of the filesystem: a **directory per state**
(`pending/`, `processing/`, `done/`, `failed/`), and `rename()` to move a
job between states. `rename()` on the same filesystem is atomic on POSIX
systems, which is exactly the property a queue needs — two workers can't
both grab the same job, because only one `rename()` can succeed.

```php
<?php
// FileQueue.php
declare(strict_types=1);

final class FileQueue
{
    public function __construct(private string $dir)
    {
        foreach (['pending', 'processing', 'done', 'failed'] as $sub) {
            $path = "$this->dir/$sub";
            if (!is_dir($path)) {
                mkdir($path, 0777, true);
            }
        }
    }

    public function push(string $jobType, array $payload): string
    {
        $id = bin2hex(random_bytes(8));
        $job = json_encode(['id' => $id, 'type' => $jobType, 'payload' => $payload, 'attempts' => 0]);
        file_put_contents("$this->dir/pending/$id.json", $job);
        return $id;
    }

    /** Atomically claim the oldest pending job, or null if the queue is empty. */
    public function pop(): ?array
    {
        $files = glob("$this->dir/pending/*.json");
        if (!$files) {
            return null;
        }
        usort($files, fn($a, $b) => filemtime($a) <=> filemtime($b));
        $file = $files[0];
        $dest = "$this->dir/processing/" . basename($file);

        if (!rename($file, $dest)) {
            return null; // another worker won the race for this job
        }
        return json_decode(file_get_contents($dest), true);
    }

    public function complete(array $job): void
    {
        $from = "$this->dir/processing/{$job['id']}.json";
        rename($from, "$this->dir/done/{$job['id']}.json");
    }

    public function fail(array $job, string $reason): void
    {
        $job['attempts']++;
        $job['error'] = $reason;
        $from = "$this->dir/processing/{$job['id']}.json";
        file_put_contents($from, json_encode($job));

        $target = $job['attempts'] < 3
            ? "$this->dir/pending/{$job['id']}.json"   // retry
            : "$this->dir/failed/{$job['id']}.json";   // give up
        rename($from, $target);
    }
}
```

## Producer and worker

```php
<?php
// demo.php
declare(strict_types=1);
require __DIR__ . '/FileQueue.php';

$dir = sys_get_temp_dir() . '/php_queue_demo';
$queue = new FileQueue($dir);

// --- producer: a web request enqueues work and returns immediately ---
$id1 = $queue->push('send_welcome_email', ['to' => 'ada@example.com']);
$id2 = $queue->push('resize_image', ['path' => '/uploads/photo.jpg']);
echo "Enqueued $id1 and $id2\n";
echo "Pending: " . count(glob("$dir/pending/*.json")) . "\n";

// --- worker: a separate process pulls jobs and executes them ---
while ($job = $queue->pop()) {
    echo "Processing job {$job['id']} of type {$job['type']}...\n";

    if ($job['type'] === 'resize_image') {
        // simulate a transient failure (e.g. ImageMagick unavailable)
        $queue->fail($job, 'ImageMagick timeout');
        echo "  failed (attempt {$job['attempts']}) -- requeued for retry\n";
        continue;
    }

    usleep(50_000); // simulate the actual work
    $queue->complete($job);
    echo "  done\n";
}

echo "Pending after worker pass: " . count(glob("$dir/pending/*.json")) . "\n";
echo "Done: " . count(glob("$dir/done/*.json")) . ", Failed: " . count(glob("$dir/failed/*.json")) . "\n";
```

```text
Enqueued ef0a436920ec778a and 80941d6c2cf16702
Pending: 2
Processing job 80941d6c2cf16702 of type resize_image...
  failed (attempt 1) -- requeued for retry
Processing job 80941d6c2cf16702 of type resize_image...
  failed (attempt 2) -- requeued for retry
Processing job 80941d6c2cf16702 of type resize_image...
  failed (attempt 3) -- requeued for retry
Processing job ef0a436920ec778a of type send_welcome_email...
  done
Pending after worker pass: 0
Done: 1, Failed: 1
```

The worker keeps re-picking `resize_image` off `pending/` because `pop()`
always grabs the oldest pending job — after three failed attempts it lands
in `failed/` instead of being requeued again, and the loop finally reaches
the email job. This is exactly the retry-with-a-cap behavior a real queue
gives you, just visible because everything runs in one process here.

## Why a real broker matters at scale

The file queue demonstrates the concepts, but has real limits: `glob()`
listing every pending file doesn't scale past a few thousand jobs, there's
no priority ordering, and nothing wakes a worker up — it has to poll. Redis
(via a `LPUSH`/`BRPOP` list, or `redis-queue`-style libraries) and RabbitMQ
(via AMQP, with `php-amqplib`) solve this: workers block efficiently until a
job arrives, brokers support priorities and delayed delivery, and a message
survives a worker crash because it isn't acknowledged (removed from the
queue) until the job actually completes — not just when it's picked up.
That "ack after success, not after pop" distinction is the difference
between **at-least-once delivery** (a crashed worker's job gets redelivered
to someone else) and jobs silently vanishing if a worker dies mid-task.

## PHP traps

**`glob()` is not atomic with the filesystem changing underneath it.** Two
workers can both list the same pending file in the split second before
either calls `rename()` — that's fine here, because `rename()` itself is
the atomic operation that decides the winner, and the loser's `pop()`
returns `null` and simply tries again. The mistake would be checking
`is_file()` and *then* reading the file as two separate steps — by the time
you read it, another worker may have already claimed and deleted it.

**Losing jobs on `unlink()` instead of `rename()`.** A queue implementation
that deletes the pending file, runs the job, and only writes to `done/` if
it succeeds loses the job entirely if the process crashes between the
delete and the write. Moving the file (`rename()`) instead of deleting it
means the job's state is always recoverable from disk, even after a crash —
this is the file-based equivalent of a broker's "don't ack until done."

**Retries without a cap create an infinite loop of pain.** A transient
failure (network blip) deserves a retry; a permanent one (malformed
payload) does not, and retrying it forever just burns CPU. `attempts < 3`
above is the minimum viable version of what production queues call a
**dead-letter queue** — a place failed jobs land for a human to inspect,
instead of disappearing or looping forever.

## Queues cheat sheet

| Concept | This module's file queue | Redis / RabbitMQ |
|---|---|---|
| Enqueue | `push()` writes a JSON file to `pending/` | `LPUSH` / `basic_publish` |
| Claim a job | `pop()` + atomic `rename()` | `BRPOP` (blocking) / consumer callback |
| Acknowledge success | move to `done/` | `basic_ack` |
| Retry | move back to `pending/`, increment `attempts` | requeue / redelivery on nack |
| Give up | move to `failed/` | dead-letter queue/exchange |
| Worker wakes on new job | polling (must check repeatedly) | blocking pop / push notification |

## Exercise

Add a `FileQueue::stats(): array` method returning
`['pending' => n, 'processing' => n, 'done' => n, 'failed' => n]` by counting
files in each directory. Then write a script that pushes five jobs of type
`'flaky_job'`, runs a worker loop where `flaky_job` fails on its first two
attempts but succeeds on the third (track attempt count in the payload, e.g.
`push('flaky_job', ['succeed_on_attempt' => 3])`), and prints `stats()`
before the run and after, confirming all five end up in `done`.
