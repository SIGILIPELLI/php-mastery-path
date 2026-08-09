# 04 · Testing Advanced (Mocking)

[Level 2's PHPUnit module](../level-2/05-phpunit-testing.md) tested pure
functions — give it inputs, assert on the return value. Real services aren't
pure: they charge payment gateways, send email, and write to databases. You
can't call a real payment API from a test suite, and you don't want a test
run to email actual customers. **Test doubles** stand in for those
collaborators so the class under test can be exercised in isolation.

## The setup: a service with two collaborators

Both dependencies are **interfaces**, injected through the constructor —
that's what makes them replaceable at test time (see
[Dependency Injection](08-dependency-injection.md)).

```php
<?php
// src/PaymentGateway.php
declare(strict_types=1);

namespace App;

interface PaymentGateway
{
    /** @return string transaction id */
    public function charge(string $customerId, int $cents): string;
}
```

```php
<?php
// src/Mailer.php
declare(strict_types=1);

namespace App;

interface Mailer
{
    public function send(string $to, string $subject): void;
}
```

```php
<?php
// src/OrderService.php
declare(strict_types=1);

namespace App;

final class OrderService
{
    public function __construct(
        private PaymentGateway $gateway,
        private Mailer $mailer,
    ) {}

    public function placeOrder(string $customerId, int $cents): string
    {
        if ($cents <= 0) {
            throw new \InvalidArgumentException("Order total must be positive");
        }

        $txId = $this->gateway->charge($customerId, $cents);
        $this->mailer->send($customerId, "Receipt for $txId");

        return $txId;
    }
}
```

## Stubs vs. mocks — the distinction that actually matters

A **stub** provides canned answers so the code under test can run. A
**mock** additionally carries an *expectation* about how it will be called,
and PHPUnit verifies that expectation automatically at the end of the test.
`createStub()` gives you the former; `createMock()` gives you an object that
can do both.

```php
<?php
// tests/OrderServiceTest.php
declare(strict_types=1);

namespace Tests;

use App\Mailer;
use App\OrderService;
use App\PaymentGateway;
use PHPUnit\Framework\TestCase;

final class OrderServiceTest extends TestCase
{
    public function testReturnsTransactionIdFromGateway(): void
    {
        // A STUB: canned answers, no expectations about how it is called.
        $gateway = $this->createStub(PaymentGateway::class);
        $gateway->method("charge")->willReturn("tx_777");

        $service = new OrderService($gateway, $this->createStub(Mailer::class));

        $this->assertSame("tx_777", $service->placeOrder("cust_1", 2500));
    }

    public function testChargesTheGatewayExactlyOnceWithTheRightArguments(): void
    {
        // A MOCK: the assertion IS the expectation, verified at teardown.
        $gateway = $this->createMock(PaymentGateway::class);
        $gateway->expects($this->once())
            ->method("charge")
            ->with("cust_1", 2500)
            ->willReturn("tx_777");

        $service = new OrderService($gateway, $this->createStub(Mailer::class));
        $service->placeOrder("cust_1", 2500);
    }

    public function testEmailsAReceiptContainingTheTransactionId(): void
    {
        $gateway = $this->createStub(PaymentGateway::class);
        $gateway->method("charge")->willReturn("tx_777");

        $mailer = $this->createMock(Mailer::class);
        $mailer->expects($this->once())
            ->method("send")
            ->with("cust_1", $this->stringContains("tx_777"));

        (new OrderService($gateway, $mailer))->placeOrder("cust_1", 2500);
    }

    public function testDoesNotChargeWhenTheTotalIsInvalid(): void
    {
        $gateway = $this->createMock(PaymentGateway::class);
        $gateway->expects($this->never())->method("charge");

        $service = new OrderService($gateway, $this->createStub(Mailer::class));

        $this->expectException(\InvalidArgumentException::class);
        $service->placeOrder("cust_1", 0);
    }

    public function testPropagatesGatewayFailures(): void
    {
        $gateway = $this->createStub(PaymentGateway::class);
        $gateway->method("charge")->willThrowException(new \RuntimeException("card declined"));

        $mailer = $this->createMock(Mailer::class);
        $mailer->expects($this->never())->method("send");   // no receipt for a failed charge

        $this->expectExceptionMessage("card declined");
        (new OrderService($gateway, $mailer))->placeOrder("cust_1", 2500);
    }
}
```

```bash
vendor/bin/phpunit tests --testdox
```

```text
PHPUnit 11.5.56 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.5.9

.....                                                               5 / 5 (100%)

Time: 00:00.009, Memory: 8.00 MB

Order Service (Tests\OrderService)
 ✔ Returns transaction id from gateway
 ✔ Charges the gateway exactly once with the right arguments
 ✔ Emails a receipt containing the transaction id
 ✔ Does not charge when the total is invalid
 ✔ Propagates gateway failures

OK (5 tests, 11 assertions)
```

Notice `testChargesTheGatewayExactlyOnceWithTheRightArguments()` has no
`assert*()` call in its body at all — yet it counts assertions. The
`expects()` chain *is* the assertion, checked when the test finishes.
`willThrowException()` is how you simulate the unhappy path (a declined
card, a network timeout) without an unreliable real service.

## Scripting stub return values

`willReturn()` answers every call identically. Three variants cover the
cases where that isn't enough:

```php
<?php
// tests/StubStylesTest.php  (excerpt)
public function testWillReturnMapPicksAnAnswerPerArgumentSet(): void
{
    $repo = $this->createStub(UserRepository::class);
    $repo->method("findEmail")->willReturnMap([
        [1, "ada@example.com"],   // last element is the RETURN value
        [2, "grace@example.com"],
    ]);

    $this->assertSame("ada@example.com", $repo->findEmail(1));
    $this->assertSame("grace@example.com", $repo->findEmail(2));
    $this->assertNull($repo->findEmail(99));   // unmapped -> default return value
}

public function testWillReturnOnConsecutiveCallsScriptsASequence(): void
{
    $gateway = $this->createStub(PaymentGateway::class);
    $gateway->method("charge")->willReturnOnConsecutiveCalls("tx_1", "tx_2");

    $this->assertSame("tx_1", $gateway->charge("c", 100));
    $this->assertSame("tx_2", $gateway->charge("c", 100));
}

public function testWillReturnCallbackComputesFromTheArguments(): void
{
    $gateway = $this->createStub(PaymentGateway::class);
    $gateway->method("charge")->willReturnCallback(
        fn(string $customerId, int $cents) => "tx_{$customerId}_{$cents}"
    );

    $this->assertSame("tx_cust_9_500", $gateway->charge("cust_9", 500));
}
```

## Fakes — when a hand-written double beats a generated one

A **fake** is a real, working implementation that takes a shortcut (memory
instead of a database). It's the right tool when the collaborator has
*state* — scripting a stub for "save then read back the new value" quickly
becomes unreadable.

```php
<?php
// tests/FakeRepositoryTest.php
declare(strict_types=1);

namespace Tests;

use App\UserRepository;
use PHPUnit\Framework\TestCase;

// A FAKE: a real, working implementation with a shortcut (memory, not a DB).
final class InMemoryUserRepository implements UserRepository
{
    /** @var array<int, string> */
    private array $rows = [];

    public function findEmail(int $userId): ?string
    {
        return $this->rows[$userId] ?? null;
    }

    public function save(int $userId, string $email): void
    {
        $this->rows[$userId] = $email;
    }
}

final class FakeRepositoryTest extends TestCase
{
    public function testFakeBehavesLikeTheRealThing(): void
    {
        $repo = new InMemoryUserRepository();

        $this->assertNull($repo->findEmail(1));
        $repo->save(1, "ada@example.com");
        $this->assertSame("ada@example.com", $repo->findEmail(1));

        $repo->save(1, "ada2@example.com");     // a stub would need re-scripting here
        $this->assertSame("ada2@example.com", $repo->findEmail(1));
    }
}
```

```text
.........                                                           9 / 9 (100%)

OK (9 tests, 20 assertions)
```

## PHP traps

**A mock can fail a test with no `assert*()` in sight.** Unmet `expects()`
expectations surface at teardown, so the reported failure points at the
*production* line that made the unexpected call:

```text
1) Tests\FailingExpectationTest::testMailerIsNeverCalled
App\Mailer::send('cust_1', 'Receipt for tx_1'): void was not expected to be called.

.../src/OrderService.php:20
.../demo/FailingExpectationTest.php:22
```

**`final` classes cannot be doubled.** PHPUnit generates doubles by
subclassing, so `final` is a hard stop:

```text
PHPUnit\Framework\MockObject\Generator\ClassIsFinalException:
Class "App\SystemClock" is declared "final" and cannot be doubled
```

The fix is not to drop `final` — it's to depend on an *interface*
(`Clock`) and let `SystemClock` stay `final`. The same applies to `static`
methods and `readonly` properties: code that reaches for them directly is
code you cannot substitute. Untestable code is usually a design signal, not
a tooling limitation.

**Over-mocking makes tests brittle.** A test that asserts on five
`expects()` chains is pinned to the *implementation*, and will break when
you refactor even though behavior is unchanged. Mock at the boundary
(network, filesystem, clock, database) — not at every internal seam.

## Test double cheat sheet

| Call | What you get |
|------|--------------|
| `$this->createStub(I::class)` | Canned answers only — never fails a test on its own |
| `$this->createMock(I::class)` | Stub plus verifiable `expects()` expectations |
| `->method("m")->willReturn($v)` | Same answer every call |
| `->willReturnMap([[$a, $r], ...])` | Answer chosen by argument values (last element = return) |
| `->willReturnOnConsecutiveCalls($a, $b)` | A different answer per call, in order |
| `->willReturnCallback($fn)` | Answer computed from the arguments |
| `->willThrowException(new E())` | Simulate the failure path |
| `->expects($this->once())` | Must be called exactly once |
| `->expects($this->never())` | Must never be called |
| `->with($a, $b)` | Constrain the arguments (matchers like `$this->stringContains()` allowed) |
| Hand-written fake | Working implementation with a shortcut — best for stateful collaborators |

## Exercise

Add an `InventoryService` interface with `reserve(string $sku, int $qty): bool`
and inject it into `OrderService` so `placeOrder()` reserves stock *before*
charging. Then write four tests: (1) a stub returning `true` proves the happy
path still returns the transaction id; (2) a mock with
`expects($this->once())->with("WIDGET", 1)` verifies the reservation
arguments; (3) a stub returning `false` proves the gateway is
`expects($this->never())` charged when stock is unavailable; (4) replace the
mailer with a hand-written fake that records sent subjects in an array, and
assert the receipt subject contains the transaction id. Run
`vendor/bin/phpunit tests --testdox` and confirm all four pass.
