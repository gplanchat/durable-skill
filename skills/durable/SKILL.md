---
name: durable
description: >
  Write workflows, activities and Nexus operations with gplanchat/durable, the PHP
  durable-execution library (Symfony, Sylius, Laravel, Magento, or no framework at all).
  Use whenever the task involves a durable workflow, a workflow method, an activity
  contract or handler, a child workflow, a timer, a signal, an update, a Nexus service
  or a Nexus operation — writing one, reviewing one, or debugging one. Triggers on
  "durable workflow", "AsWorkflowMethod", "AsActivityMethod", "AsNexusOperation",
  "activityStub", "childWorkflowStub", "nexusStub", "WorkflowEnvironment",
  "continueAsNew", "durable-bundle", "durable-plugin", "durable-magento",
  "durable-laravel", and on questions about determinism, replay, retries or
  compensation in PHP durable execution. Also covers **migrating** existing code with
  `gplanchat/durable-rector` — the `temporal-sdk` and `durable-upgrade` sets — and
  finishing by hand what the rules deliberately leave alone.
---

# Writing Durable code

A workflow is a **plain PHP class**. No base class, no interface, no framework type in
its signature. What makes it durable is that the engine replays it from a journal, so
the only rule that really binds is determinism.

## The three shapes

```php
#[AsWorkflow('checkout')]                       // the name the cluster knows
final class CheckoutWorkflow
{
    public function __construct(
        private readonly WorkflowEnvironment $environment,
    ) {}

    #[AsWorkflowMethod]                          // exactly one per workflow class
    public function run(string $orderId): string
    {
        $orders = $this->environment->activityStub(OrderActivities::class);

        $receipt = $this->environment->await($orders->charge($orderId));
        $this->environment->sleep(Duration::seconds(30), 'cooling down');

        return $this->environment->await($orders->notify($receipt));
    }
}
```

```php
interface OrderActivities                        // the CONTRACT carries the names
{
    #[AsActivityMethod(name: 'shop.order.charge')]
    public function charge(string $orderId): string;
}

final class DoctrineOrderActivities implements OrderActivities   // the HANDLER does the work
{
    public function charge(string $orderId): string { /* real side effects here */ }
}
```

```php
#[AsNexusService('billing')]                     // another team, another deployment
interface BillingContract
{
    #[AsNexusOperation('check')]
    public function check(string $order, int $amountInCents, string $currency): array;
}
```

**Never write an activity name as a string at the call site.** The stub reads the
contract's attributes, so the name lives in exactly one place. This is the single
most common way to break an existing workflow: rename the method, keep the attribute,
and running executions still find their activity.

## The rule that binds: the workflow body is replayed

Everything in a `#[AsWorkflowMethod]` runs again, from the top, every time the engine
resumes the execution — possibly in another process, days later. So the body must give
the same answers when replayed:

| Never in a workflow body | Use instead |
|---|---|
| `rand()`, `uniqid()`, `time()`, `new \DateTime()` | `$environment->sideEffect(fn() => …)` — the result is journaled once |
| `sleep()`, `usleep()` | `$environment->sleep($duration, 'why we wait')` |
| A database query, an HTTP call, a file write | an **activity** |
| `if (getenv(…))` to branch on a deploy | `$environment->version($changeId, $min, $max)` |

An **activity** is where non-determinism belongs. It runs once per attempt, its result
is written to the journal, and on replay the journal answers instead of the code.

⚠ **An activity is retried; write it so a second attempt is safe.** The engine
guarantees the *result* is recorded once, not that your side effect is. Charging a card
twice is a business incident, not a framework bug.

## Waiting

Everything schedulable returns an `Awaitable`. Nothing happens until you await it —
which is also how you get parallelism:

```php
$a = $stub->reserve($order);                     // scheduled, not awaited
$b = $stub->quote($order);
[$reserved, $quoted] = $this->environment->await($this->environment->all($a, $b));

$first = $this->environment->await($this->environment->any($a, $b));
$twoOfThree = $this->environment->await($this->environment->some(2, $a, $b, $c));
```

`await()` takes an optional deadline: `await($awaitable, Duration::minutes(5))`.

## Reference files

Read the one you need, not all three:

- `references/workflows.md` — signals, updates, child workflows, timers,
  `continueAsNew`, versioning, and how an execution is *started* (never inline in a web
  request).
- `references/activities.md` — `ActivityOptions`, retry policy, the four timeouts,
  non-retryable exceptions, heartbeats, cancellation.
- `references/nexus.md` — Nexus contracts, why a contract splits in two when a
  workflow fulfils an operation, and the payload trap that fails silently.
- `references/migrations.md` — **read this before touching existing code.** Coming off
  the Temporal PHP SDK, or moving from one Durable version to the next: `gplanchat/durable-rector`
  does the rewriting, and the reference is mostly about the five things its rules refuse
  to guess. Two mistakes raise no error on a server that is already running: a workflow
  or activity type name that changes, so runs in flight stop resolving, and a `catch` on
  `ApplicationFailure`, `ServerFailure`, `TerminatedFailure` or `TimeoutFailure`, SDK
  failures with no Durable counterpart, which never matches after the migration. The
  migration marks every reference to those four failures (a `catch`, a parameter or return
  type included) and every activity name it cannot carry over with a `durable-rector:`
  comment. A workflow name gets no marker when the SDK attribute sits on
  the class itself, when Rector cannot load the class, or when the class already carries
  `#[AsWorkflow]`, which keeps its name whatever the SDK type was: check those by hand.

## Before you hand the code back

- One `#[AsWorkflowMethod]` per workflow class, and the class is declared to the host
  (Symfony/Sylius autoconfigure it; Magento lists it in `di.xml`).
- Every activity call goes through a stub built from a contract — no string names.
- Nothing non-deterministic in the workflow body.
- Every timer has a summary: it is what names the row in the observation timeline, and
  `TimerScheduled` tells an operator nothing at 3am.
- The workflow does not catch what it cannot compensate. An unhandled failure ends the
  execution and is visible; a swallowed one leaves an order half-processed and silent.
