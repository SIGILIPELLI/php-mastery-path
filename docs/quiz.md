---
title: "Final Quiz"
description: "Test what you have learned across all four levels of PHP Mastery Path with a 18-question final quiz."
---

# Final Quiz

18 questions drawn from all four levels of this course. You need **70% (13 out of 18)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does `==` do with \"5\" and 5 compared to `===`?",
    "options": [
      "Reverse",
      "== coerces types and is true; === also checks type and is false",
      "Both false",
      "Both true"
    ],
    "correctIndex": 1,
    "explanation": "Prefer ===."
  },
  {
    "question": "What does this print?\n```php\n$a = [\"x\" => null];\nvar_dump(isset($a[\"x\"]), array_key_exists(\"x\", $a), $a[\"y\"] ?? \"d\");\n```\n",
    "options": [
      "bool(false)\nbool(true)\nstring(1) \"d\"",
      "bool(true)\nbool(true)\nstring(1) \"d\"",
      "bool(false)\nbool(false)\nNULL",
      "Error"
    ],
    "correctIndex": 0,
    "explanation": "isset treats null as unset. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```php\nfunction f(int ...$n) { return array_sum($n); }\n$args = [1, 2, 3];\necho f(...$args), \" \", f(), \" \", (fn($x) => $x ** 2)(5);\n```\n",
    "options": [
      "6 0 25",
      "Error",
      "6 null 10",
      "123 0 25"
    ],
    "correctIndex": 0,
    "explanation": "Spread and variadics. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```php\n$n = 7;\necho match(true) { $n % 2 === 0 => \"even\", $n > 5 => \"big odd\", default => \"odd\" };\ntry { echo match($n) { 1 => \"a\" }; } catch (\\UnhandledMatchError $e) { echo \" \", get_class($e); }\n```\n",
    "options": [
      "big odd null",
      "even UnhandledMatchError",
      "big odd UnhandledMatchError",
      "odd"
    ],
    "correctIndex": 2,
    "explanation": "match is exhaustive. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```php\nclass P { public function __construct(public readonly int $x = 1) {} }\n$p = new P(5);\ntry { $p->x = 9; } catch (Error $e) { echo get_class($e), \" \", $e->getMessage(); }\n```\n",
    "options": [
      "TypeError",
      "Error 9",
      "No output",
      "Error Cannot modify readonly property P::$x"
    ],
    "correctIndex": 3,
    "explanation": "Readonly properties are immutable. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```php\nclass A { public static function make() { return new static; } public function who() { return static::class; } }\nclass B extends A {}\necho (B::make())->who(), \" \", (new A)->who();\n```\n",
    "options": [
      "Error",
      "B B",
      "A A",
      "B A"
    ],
    "correctIndex": 3,
    "explanation": "Late static binding. Verified by compiling and running it."
  },
  {
    "question": "When prefer an interface to an abstract class?",
    "options": [
      "Never",
      "For speed",
      "For state",
      "To define a contract that unrelated classes can implement"
    ],
    "correctIndex": 3,
    "explanation": "Composition."
  },
  {
    "question": "What does this print?\n```php\nfunction f() { try { return \"try\"; } finally { echo \"[cleanup]\"; } }\necho f();\n```\n",
    "options": [
      "try",
      "try[cleanup]",
      "[cleanup]",
      "[cleanup]try"
    ],
    "correctIndex": 3,
    "explanation": "finally runs before the value is returned. Verified by compiling and running it."
  },
  {
    "question": "Why are prepared statements the main defence against SQL injection?",
    "options": [
      "User input is sent as data, never parsed as SQL",
      "They are faster",
      "They are encrypted",
      "They escape quotes"
    ],
    "correctIndex": 0,
    "explanation": "Bind parameters."
  },
  {
    "question": "What does this print?\n```php\n$pdo = new PDO(\"sqlite::memory:\");\n$pdo->exec(\"CREATE TABLE t(id INTEGER PRIMARY KEY, n TEXT)\");\n$s = $pdo->prepare(\"INSERT INTO t(n) VALUES(:n)\");\n$s->execute([\":n\" => \"a\"]); $s->execute([\":n\" => \"b\"]);\necho $pdo->lastInsertId(), \" \", json_encode($pdo->query(\"SELECT * FROM t\")->fetchAll(PDO::FETCH_ASSOC));\n```\n",
    "options": [
      "1 []",
      "2 [{\"id\":1,\"n\":\"a\"},{\"id\":2,\"n\":\"b\"}]",
      "2 [{\"id\":\"1\"}]",
      "Error"
    ],
    "correctIndex": 1,
    "explanation": "Named parameters and fetch modes. Verified by compiling and running it."
  },
  {
    "question": "What should happen to the session id after login?",
    "options": [
      "Put it in the URL",
      "Keep it",
      "Regenerate it with session_regenerate_id(true)",
      "Echo it"
    ],
    "correctIndex": 2,
    "explanation": "Prevents session fixation."
  },
  {
    "question": "Which function stores a password?",
    "options": [
      "sha1()",
      "password_hash(), verified with password_verify()",
      "md5()",
      "base64_encode()"
    ],
    "correctIndex": 1,
    "explanation": "Adaptive hashing."
  },
  {
    "question": "What does this print?\n```php\necho htmlspecialchars(\"<b>\\\"Tom\\\" & 'Jerry'<\/b>\"), \" \", strlen(\"日本\"), \" \", mb_strlen(\"日本\");\n```\n",
    "options": [
      "&lt;b&gt;&quot;Tom&quot; &amp; &#039;Jerry&#039;&lt;/b&gt; 6 2",
      "<b>\"Tom\" & 'Jerry'<\/b> 2 2",
      "&lt;b&gt; 6 6",
      "Error"
    ],
    "correctIndex": 0,
    "explanation": "Escape output; strlen counts bytes. Verified by compiling and running it."
  },
  {
    "question": "What does PSR-4 map?",
    "options": [
      "URLs to routes",
      "Tables to classes",
      "Functions to files",
      "Namespace prefixes to directories so classes autoload"
    ],
    "correctIndex": 3,
    "explanation": "Composer autoload."
  },
  {
    "question": "What do you commit for an application's Composer dependencies?",
    "options": [
      "Only composer.json",
      "Nothing",
      "vendor/",
      "composer.lock, so every environment installs the same versions"
    ],
    "correctIndex": 3,
    "explanation": "Reproducible builds."
  },
  {
    "question": "A test needs a repository without a database. What is a good approach?",
    "options": [
      "Skip the test",
      "Use globals",
      "Use prod DB",
      "Inject an in-memory fake implementing the repository interface"
    ],
    "correctIndex": 3,
    "explanation": "Dependency injection."
  },
  {
    "question": "Why queue slow work such as sending email?",
    "options": [
      "Keeps requests fast and allows retries outside the request",
      "Shorter code",
      "Style",
      "Law"
    ],
    "correctIndex": 0,
    "explanation": "Handlers must be idempotent."
  },
  {
    "question": "The app is slow under load. What first?",
    "options": [
      "Add random caches",
      "Upgrade PHP only",
      "Rewrite in C",
      "Profile to find the real hotspot (often N+1 queries or missing indexes)"
    ],
    "correctIndex": 3,
    "explanation": "Measure first."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A shopping cart with discounts

Implement Cart: add (merging quantities and rejecting non-positive quantities with InvalidArgumentException), remove, subtotal, a 10% discount when the subtotal is at least 100.00, total and count. Work in integer cents to avoid float errors.

**Starter code**

```php
final class Cart {
    private array $lines = [];
    public function add(string $sku, int $qty, int $priceCents): void {
        // throw InvalidArgumentException("qty must be positive") when $qty <= 0
        // merge quantities when the same SKU is added again
    }
    public function remove(string $sku): void { }
    public function subtotal(): int { return 0; }          // sum of qty * price in cents
    public function discount(): int { return 0; }          // 10% (integer cents) when subtotal >= 10000
    public function total(): int { return 0; }
    public function count(): int { return 0; }            // total number of items
}
function money(int $cents): string { return number_format($cents / 100, 2); }

$c = new Cart();
$c->add("pen", 3, 250);
$c->add("book", 2, 4500);
$c->add("pen", 2, 250);
echo "items: ", $c->count(), "\n";
echo "subtotal: ", money($c->subtotal()), "\n";
echo "discount: ", money($c->discount()), "\n";
echo "total: ", money($c->total()), "\n";
$c->remove("book");
echo "after remove: ", money($c->total()), "\n";
try { $c->add("x", 0, 100); } catch (InvalidArgumentException $e) { echo "error: ", $e->getMessage(), "\n"; }
```

**Expected output**

```text
items: 7
subtotal: 102.50
discount: 10.25
total: 92.25
after remove: 12.50
error: qty must be positive
```

<details><summary>Show solution and explanation</summary>

<pre><code>final class Cart {
    private array $lines = [];
    public function add(string $sku, int $qty, int $priceCents): void {
        if ($qty &lt;= 0) throw new InvalidArgumentException(&quot;qty must be positive&quot;);
        if (isset($this-&gt;lines[$sku])) { $this-&gt;lines[$sku][&#x27;qty&#x27;] += $qty; }
        else { $this-&gt;lines[$sku] = [&#x27;qty&#x27; =&gt; $qty, &#x27;price&#x27; =&gt; $priceCents]; }
    }
    public function remove(string $sku): void { unset($this-&gt;lines[$sku]); }
    public function subtotal(): int {
        return array_sum(array_map(fn($l) =&gt; $l[&#x27;qty&#x27;] * $l[&#x27;price&#x27;], $this-&gt;lines));
    }
    public function discount(): int {
        $s = $this-&gt;subtotal();
        return $s &gt;= 10000 ? intdiv($s * 10, 100) : 0;
    }
    public function total(): int { return $this-&gt;subtotal() - $this-&gt;discount(); }
    public function count(): int { return array_sum(array_column($this-&gt;lines, &#x27;qty&#x27;)); }
}
function money(int $cents): string { return number_format($cents / 100, 2); }

$c = new Cart();
$c-&gt;add(&quot;pen&quot;, 3, 250);
$c-&gt;add(&quot;book&quot;, 2, 4500);
$c-&gt;add(&quot;pen&quot;, 2, 250);
echo &quot;items: &quot;, $c-&gt;count(), &quot;\n&quot;;
echo &quot;subtotal: &quot;, money($c-&gt;subtotal()), &quot;\n&quot;;
echo &quot;discount: &quot;, money($c-&gt;discount()), &quot;\n&quot;;
echo &quot;total: &quot;, money($c-&gt;total()), &quot;\n&quot;;
$c-&gt;remove(&quot;book&quot;);
echo &quot;after remove: &quot;, money($c-&gt;total()), &quot;\n&quot;;
try { $c-&gt;add(&quot;x&quot;, 0, 100); } catch (InvalidArgumentException $e) { echo &quot;error: &quot;, $e-&gt;getMessage(), &quot;\n&quot;; }
</code></pre>

<p>Money is kept in integer cents so arithmetic is exact; array_sum/array_map compute totals and unset removes a line. Verified by running the solution with PHP.</p>
</details>

### A router with a middleware pipeline

Implement Router::dispatch so a request passes through the registered middleware (first registered runs outermost) before reaching the route handler, returning 404 for unknown paths. Auth short-circuits with 401 when the token is wrong, and Logger records the request and response status.

**Starter code**

```php
interface Middleware { public function handle(array $req, callable $next): array; }
final class Auth implements Middleware {
    public function handle(array $req, callable $next): array {
        if (($req['token'] ?? '') !== 'secret') return ['status' => 401, 'body' => 'unauthorized'];
        return $next($req);
    }
}
final class Logger implements Middleware {
    public static array $log = [];
    public function handle(array $req, callable $next): array {
        self::$log[] = "-> {$req['path']}";
        $res = $next($req);
        self::$log[] = "<- {$res['status']}";
        return $res;
    }
}
final class Router {
    private array $routes = [];
    private array $mw = [];
    public function add(string $path, callable $h): void { $this->routes[$path] = $h; }
    public function use(Middleware $m): void { $this->mw[] = $m; }
    public function dispatch(array $req): array {
        // core handler: 200 + handler output, or 404 "not found"
        // wrap it with the middleware so the FIRST registered middleware runs outermost (array_reduce over the reversed list)
        return ['status' => 500, 'body' => 'todo'];
    }
}

$r = new Router();
$r->use(new Logger());
$r->use(new Auth());
$r->add("/hello", fn($req) => "hello " . ($req['name'] ?? "world"));
foreach ([
    ['path' => '/hello', 'token' => 'secret', 'name' => 'Ada'],
    ['path' => '/hello', 'token' => 'bad'],
    ['path' => '/missing', 'token' => 'secret'],
] as $req) {
    $res = $r->dispatch($req);
    echo $res['status'], " ", $res['body'], "\n";
}
echo implode(" | ", Logger::$log), "\n";
```

**Expected output**

```text
200 hello Ada
401 unauthorized
404 not found
-> /hello | <- 200 | -> /hello | <- 401 | -> /missing | <- 404
```

<details><summary>Show solution and explanation</summary>

<pre><code>interface Middleware { public function handle(array $req, callable $next): array; }
final class Auth implements Middleware {
    public function handle(array $req, callable $next): array {
        if (($req[&#x27;token&#x27;] ?? &#x27;&#x27;) !== &#x27;secret&#x27;) return [&#x27;status&#x27; =&gt; 401, &#x27;body&#x27; =&gt; &#x27;unauthorized&#x27;];
        return $next($req);
    }
}
final class Logger implements Middleware {
    public static array $log = [];
    public function handle(array $req, callable $next): array {
        self::$log[] = &quot;-&gt; {$req[&#x27;path&#x27;]}&quot;;
        $res = $next($req);
        self::$log[] = &quot;&lt;- {$res[&#x27;status&#x27;]}&quot;;
        return $res;
    }
}
final class Router {
    private array $routes = [];
    private array $mw = [];
    public function add(string $path, callable $h): void { $this-&gt;routes[$path] = $h; }
    public function use(Middleware $m): void { $this-&gt;mw[] = $m; }
    public function dispatch(array $req): array {
        $core = function (array $r): array {
            $h = $this-&gt;routes[$r[&#x27;path&#x27;]] ?? null;
            return $h ? [&#x27;status&#x27; =&gt; 200, &#x27;body&#x27; =&gt; $h($r)] : [&#x27;status&#x27; =&gt; 404, &#x27;body&#x27; =&gt; &#x27;not found&#x27;];
        };
        $chain = array_reduce(array_reverse($this-&gt;mw), fn($next, $m) =&gt; fn($r) =&gt; $m-&gt;handle($r, $next), $core);
        return $chain($req);
    }
}

$r = new Router();
$r-&gt;use(new Logger());
$r-&gt;use(new Auth());
$r-&gt;add(&quot;/hello&quot;, fn($req) =&gt; &quot;hello &quot; . ($req[&#x27;name&#x27;] ?? &quot;world&quot;));
foreach ([
    [&#x27;path&#x27; =&gt; &#x27;/hello&#x27;, &#x27;token&#x27; =&gt; &#x27;secret&#x27;, &#x27;name&#x27; =&gt; &#x27;Ada&#x27;],
    [&#x27;path&#x27; =&gt; &#x27;/hello&#x27;, &#x27;token&#x27; =&gt; &#x27;bad&#x27;],
    [&#x27;path&#x27; =&gt; &#x27;/missing&#x27;, &#x27;token&#x27; =&gt; &#x27;secret&#x27;],
] as $req) {
    $res = $r-&gt;dispatch($req);
    echo $res[&#x27;status&#x27;], &quot; &quot;, $res[&#x27;body&#x27;], &quot;\n&quot;;
}
echo implode(&quot; | &quot;, Logger::$log), &quot;\n&quot;;
</code></pre>

<p>array_reduce over the reversed middleware list builds nested closures (the onion model); a middleware can return early without calling $next. Verified by running the solution with PHP.</p>
</details>
