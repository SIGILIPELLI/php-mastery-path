# 09 · Working with Dates

Dates look simple until you have to add days across a month boundary, compare
two timestamps in different time zones, or format one for display. PHP's
`DateTime` family (`DateTime`, `DateTimeImmutable`, `DateInterval`,
`DateTimeZone`) handles all of that correctly — far more reliably than
hand-rolling date math with raw Unix timestamps and `86400`-seconds-per-day
assumptions (which break on daylight saving transitions).

## Creating dates

```php
<?php

$now = new DateTime();                          // right now, server's default timezone
$specific = new DateTime("2026-03-15 14:30:00"); // a specific moment
$fromNow = new DateTime("+3 days");              // PHP parses relative expressions too

echo $now->format("Y-m-d H:i:s") . "\n";
echo $specific->format("Y-m-d H:i:s") . "\n";    // 2026-03-15 14:30:00
echo $fromNow->format("Y-m-d") . "\n";
```

`new DateTime("...")` accepts a wide range of English-language date formats
("next Monday", "first day of next month", "-2 weeks") in addition to
ISO-style strings — convenient, but for anything parsed from user input or
an API, prefer the strict parsing shown below instead of relying on this
guesswork.

## `DateTime` vs `DateTimeImmutable`: a common trap

`DateTime` objects are **mutable** — methods like `modify()` and `add()`
change the object in place and also return it, which is easy to misuse when
you only meant to compute a new value.

```php
<?php

$original = new DateTime("2026-01-01");
$modified = $original->modify("+1 month");

// SURPRISE: both variables point at the SAME object -- modify() changed
// $original in place and returned the same instance, it did NOT create a
// separate "one month later" copy.
echo $original->format("Y-m-d") . "\n";   // 2026-02-01 -- not "2026-01-01"!
echo $modified->format("Y-m-d") . "\n";   // 2026-02-01
```

```php
<?php

// DateTimeImmutable avoids the trap entirely: every "modifying" method
// returns a NEW instance and leaves the original untouched.
$original = new DateTimeImmutable("2026-01-01");
$modified = $original->modify("+1 month");

echo $original->format("Y-m-d") . "\n";   // 2026-01-01 -- unchanged
echo $modified->format("Y-m-d") . "\n";   // 2026-02-01 -- a separate object
```

Prefer `DateTimeImmutable` by default in new code — it behaves the way most
people already assume `DateTime` works, and eliminates a class of bugs where
a date gets silently changed somewhere deep in a function that only looked
like it was reading the value.

## Formatting dates

```php
<?php

$date = new DateTimeImmutable("2026-07-04 09:05:00");

echo $date->format("Y-m-d") . "\n";        // 2026-07-04
echo $date->format("d/m/Y") . "\n";        // 04/07/2026
echo $date->format("D, d M Y") . "\n";     // Sat, 04 Jul 2026
echo $date->format("H:i:s") . "\n";        // 09:05:00
echo $date->format("g:i A") . "\n";        // 9:05 AM
echo $date->format(DATE_ATOM) . "\n";      // 2026-07-04T09:05:00+00:00 -- ISO 8601, good for APIs
```

| Format char | Meaning | Example |
|:---:|---|---|
| `Y` | 4-digit year | `2026` |
| `m` | 2-digit month | `07` |
| `d` | 2-digit day | `04` |
| `H` | 24-hour hour | `09` |
| `g` | 12-hour hour, no leading zero | `9` |
| `i` | minutes | `05` |
| `s` | seconds | `00` |
| `D` | short day name | `Sat` |
| `M` | short month name | `Jul` |
| `A` | AM/PM | `AM` |

## Parsing strings strictly with `createFromFormat`

When the input format is known and fixed (a CSV column, a specific API
field), `createFromFormat()` parses exactly that shape and fails loudly on
anything else — safer than the loose, guessing constructor above.

```php
<?php

$input = "15/03/2026";   // day/month/year -- ambiguous without a declared format
$date = DateTimeImmutable::createFromFormat("d/m/Y", $input);

if ($date === false) {
    echo "Could not parse date: $input\n";
} else {
    echo $date->format("Y-m-d") . "\n";   // 2026-03-15
}

// Passing the same digits with the WRONG format silently misreads
// day/month instead of failing -- this is why declaring the format matters:
$wrong = DateTimeImmutable::createFromFormat("m/d/Y", $input);
// Read as month=15, day=03, year=2026. Month 15 doesn't exist, so PHP
// normalizes by overflowing: 3 months past December rolls into next year.
echo $wrong->format("Y-m-d") . "\n";   // 2027-03-03 -- silently wrong, not an error
```

## Date arithmetic with `DateInterval`

```php
<?php

$start = new DateTimeImmutable("2026-01-31");

$plusOneMonth = $start->add(new DateInterval("P1M"));   // "Period of 1 Month"
echo $plusOneMonth->format("Y-m-d") . "\n";   // 2026-03-03 -- NOT Feb 31!

$plusTenDays = $start->add(new DateInterval("P10D"));
echo $plusTenDays->format("Y-m-d") . "\n";    // 2026-02-10

$minusOneWeek = $start->sub(new DateInterval("P1W"));
echo $minusOneWeek->format("Y-m-d") . "\n";   // 2026-01-24
```

Adding "1 month" to January 31st is a well-known trap: February doesn't
have 31 days, so PHP overflows into March rather than silently clamping to
February 28th. If a calendar feature depends on month-end behavior, check
the result explicitly rather than assuming it lands where you expect.

## Comparing dates and computing a difference

`DateTime` objects compare directly with PHP's normal comparison operators,
and `diff()` produces a `DateInterval` describing the gap between two dates.

```php
<?php

$deadline = new DateTimeImmutable("2026-12-01");
$today = new DateTimeImmutable("2026-08-02");

var_dump($today < $deadline);   // bool(true) -- comparison operators just work

$interval = $today->diff($deadline);
echo "{$interval->days} days remaining\n";                 // 121 days remaining
echo "{$interval->m} months, {$interval->d} days\n";       // 3 months, 29 days
```

`$interval->days` gives the total number of whole days between the two
dates, while `$interval->m`/`$interval->d` break the same gap into calendar
months and remaining days — pick whichever matches what you're displaying.

## Time zones

```php
<?php

$utc = new DateTimeImmutable("2026-06-15 12:00:00", new DateTimeZone("UTC"));
$mumbai = $utc->setTimezone(new DateTimeZone("Asia/Kolkata"));

echo $utc->format("Y-m-d H:i:s T") . "\n";      // 2026-06-15 12:00:00 UTC
echo $mumbai->format("Y-m-d H:i:s T") . "\n";   // 2026-06-15 17:30:00 IST
```

Always store and compare dates in UTC internally (e.g. in a database), and
convert to a specific time zone only when displaying to a particular user —
mixing local time zones through your business logic is a reliable source of
off-by-one-hour (or off-by-a-day, near midnight) bugs.

## Dates cheat sheet

| Tool | Purpose |
|------|---------|
| `DateTimeImmutable` | Preferred date/time type — every operation returns a new instance |
| `DateTime` | Mutable equivalent — `modify()`/`add()`/`sub()` change the object in place |
| `->format($fmt)` | Render as a string using format characters |
| `DateTimeImmutable::createFromFormat($fmt, $str)` | Strict parsing of a known input format |
| `DateInterval` | A span of time (`P1M`, `P10D`, `P1W`) used with `add()`/`sub()` |
| `->diff($other)` | Compute the `DateInterval` between two dates |
| `DateTimeZone` | Represents a named time zone (`"UTC"`, `"Asia/Kolkata"`) |
| `->setTimezone($tz)` | Return the same instant, viewed in a different time zone |

## Exercise

Write a function `daysUntil(DateTimeImmutable $target, ?DateTimeImmutable
$from = null): int` that returns how many whole days remain until `$target`
(defaulting `$from` to "now" when not given), using `diff()`. Then write
`isWeekend(DateTimeImmutable $date): bool` using `$date->format("N")` (ISO-8601
day-of-week number, 1 = Monday through 7 = Sunday). Test `daysUntil()` against
a date in the past (it should return a negative number — check what `diff()`
actually gives you for a past date and adjust if needed) and `isWeekend()`
against a known Saturday and a known Tuesday.
