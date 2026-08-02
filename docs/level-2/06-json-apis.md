# 06 · Working with JSON/APIs

JSON is the de facto data format for web APIs — lightweight, human-readable,
and natively supported by PHP's `json_encode()`/`json_decode()` functions.
This module covers converting between PHP values and JSON, calling external
HTTP APIs to consume JSON, and building a small JSON endpoint of your own.

## Encoding PHP values to JSON

```php
<?php

$user = [
    "name" => "Ada Lovelace",
    "age" => 36,
    "active" => true,
    "tags" => ["math", "computing"],
];

$json = json_encode($user, JSON_PRETTY_PRINT);
echo $json;
// {
//     "name": "Ada Lovelace",
//     "age": 36,
//     "active": true,
//     "tags": [
//         "math",
//         "computing"
//     ]
// }
```

`json_encode()` maps PHP types onto JSON types fairly directly: arrays with
sequential integer keys become JSON arrays (`[...]`), associative arrays
become JSON objects (`{...}`), and `true`/`false`/`null` map onto their JSON
equivalents unchanged.

## Decoding JSON to PHP values

```php
<?php

$json = '{"name":"Alan Turing","age":41,"skills":["logic","cryptography"]}';

// Second argument `true` -> associative arrays instead of stdClass objects
$data = json_decode($json, true);

echo $data["name"] . "\n";        // Alan Turing
echo $data["skills"][0] . "\n";   // logic

// Without `true`, you get stdClass objects with -> access instead of ['key']
$obj = json_decode($json);
echo $obj->name . "\n";           // Alan Turing
echo $obj->skills[1] . "\n";      // cryptography
```

## Always check for decode failure

By default, `json_decode()` returns `null` on malformed input — which is
easy to confuse with a JSON payload that legitimately *is* the value `null`.
`JSON_THROW_ON_ERROR` turns a parse failure into a catchable exception
instead, so a bug can't hide behind a silent `null`.

```php
<?php

$badJson = '{"name": "Ada", }';   // trailing comma -- not valid JSON

try {
    $data = json_decode($badJson, true, flags: JSON_THROW_ON_ERROR);
} catch (JsonException $e) {
    echo "Invalid JSON: " . $e->getMessage() . "\n";
    // Invalid JSON: Syntax error
}
```

Always pass `JSON_THROW_ON_ERROR` (encoding or decoding) unless you have a
specific reason not to — the alternative is manually checking
`json_last_error() !== JSON_ERROR_NONE` after every single call, which is
easy to forget even once.

## Making objects JSON-serializable

By default, `json_encode()` on an object only picks up its **public**
properties. To control exactly what an object's JSON looks like — hiding
internals, renaming fields, computing derived values — implement
`JsonSerializable`.

```php
<?php

class Product implements JsonSerializable
{
    public function __construct(
        private string $name,
        private int $priceCents,   // stored as cents internally
    ) {}

    public function jsonSerialize(): array
    {
        return [
            "name" => $this->name,
            "price" => $this->priceCents / 100,   // exposed as dollars
        ];
    }
}

echo json_encode(new Product("Widget", 1999));
// {"name":"Widget","price":19.99}
```

Without `JsonSerializable`, `json_encode()` on this `Product` would produce
`{}` — every property is `private`, and `json_encode()` never reaches into
private/protected state on its own.

## Consuming a JSON API with `file_get_contents`

For simple `GET` requests, `file_get_contents()` with a stream context is
enough — no extra extension required.

```php
<?php

$context = stream_context_create([
    "http" => [
        "method" => "GET",
        "header" => "Accept: application/json\r\n",
        "timeout" => 5,   // seconds -- never make a blocking call with no timeout
        "ignore_errors" => true,   // still return the body on 4xx/5xx instead of false
    ],
]);

$response = file_get_contents("https://api.example.com/users/1", context: $context);

if ($response === false) {
    echo "Request failed (network error, DNS, etc.)\n";
} else {
    // $http_response_header is a special variable PHP populates automatically
    // after an HTTP stream request, containing the raw response headers.
    $statusLine = $http_response_header[0] ?? "";
    echo "Status: $statusLine\n";

    $data = json_decode($response, true, flags: JSON_THROW_ON_ERROR);
    echo "Name: " . ($data["name"] ?? "unknown") . "\n";
}
```

## Consuming a JSON API with cURL

cURL (the `ext-curl` extension) gives finer control — custom headers,
`POST`/`PUT`/`DELETE` bodies, authentication, retries — and is the more
common choice for anything beyond a quick `GET`.

```php
<?php

function fetchJson(string $url, string $method = "GET", ?array $body = null): array
{
    $ch = curl_init($url);

    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,     // return the body as a string instead of printing it
        CURLOPT_CUSTOMREQUEST => $method,
        CURLOPT_TIMEOUT => 5,
        CURLOPT_HTTPHEADER => ["Accept: application/json", "Content-Type: application/json"],
    ]);

    if ($body !== null) {
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body, JSON_THROW_ON_ERROR));
    }

    $response = curl_exec($ch);

    if ($response === false) {
        $error = curl_error($ch);
        curl_close($ch);
        throw new RuntimeException("cURL request failed: $error");
    }

    $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($status >= 400) {
        throw new RuntimeException("API returned HTTP $status");
    }

    return json_decode($response, true, flags: JSON_THROW_ON_ERROR);
}

// $user = fetchJson("https://api.example.com/users/1");
// $created = fetchJson("https://api.example.com/users", "POST", ["name" => "New User"]);
```

Checking `curl_exec()`'s return value AND the HTTP status code are both
necessary — a `false` return means the request itself never completed
(timeout, DNS failure, refused connection), while a `200`-vs-`404`-vs-`500`
status means it completed but the *server* is reporting success or failure.

## Building a small JSON API endpoint

```php
<?php
// api/user.php -- returns JSON instead of HTML
header("Content-Type: application/json");

$id = (int) ($_GET["id"] ?? 0);

$users = [1 => ["id" => 1, "name" => "Ada Lovelace"]];

if (!isset($users[$id])) {
    http_response_code(404);
    echo json_encode(["error" => "User not found"]);
    exit;
}

echo json_encode($users[$id]);
```

Setting the `Content-Type: application/json` header tells the client
(browser, `curl`, another service) how to interpret the body, and setting an
accurate `http_response_code()` lets API consumers branch on status without
having to parse the error message text.

## JSON cheat sheet

| Function / flag | Purpose |
|------------------|---------|
| `json_encode($value, $flags)` | PHP value → JSON string |
| `json_decode($json, true, flags: ...)` | JSON string → associative array |
| `json_decode($json)` (no `true`) | JSON string → `stdClass` object(s) |
| `JSON_THROW_ON_ERROR` | Throw `JsonException` instead of failing silently |
| `JSON_PRETTY_PRINT` | Human-readable, indented output |
| `JsonSerializable` | Interface to control exactly how an object encodes |
| `CURLOPT_RETURNTRANSFER` | Return the response body instead of printing it |
| `http_response_code()` | Set the HTTP status code of your own response |

## Exercise

Write a function `fetchJsonSafely(string $url): array` that wraps
`file_get_contents()` with a stream context (5-second timeout), returns
`["error" => "..."]` on a network failure, and otherwise decodes the JSON
body with `JSON_THROW_ON_ERROR`, catching `JsonException` and returning an
`["error" => "..."]` array for malformed responses too — so callers never
have to distinguish "network failed" from "bad JSON" from "success," just
check `isset($result["error"])`. Then write a small `Order` class
implementing `JsonSerializable` that stores an internal `DateTime` for
`placedAt` but exposes it as an ISO-8601 string (`$date->format(DATE_ATOM)`)
in its JSON output.
