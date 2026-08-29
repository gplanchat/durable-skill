# Activities — options, retries, timeouts

## The contract carries the name, the handler carries the work

```php
interface OrderActivities
{
    #[AsActivityMethod(name: 'shop.order.charge')]
    public function charge(string $orderId): string;
}
```

The runtime reads the handler's interfaces and keeps the methods carrying
`#[AsActivityMethod]`. You declare the handler; you never declare the names. One
declaration fewer to get wrong, and the name survives a method rename.

## Options

```php
$stub = $environment->activityStub(
    OrderActivities::class,
    ActivityOptions::of(
        retryLimit: RetryLimit::ofAttempts(3),      // total attempts, not retries
        initialInterval: 1.0,                        // seconds, or a Duration
        backoffCoefficient: 2.0,
        maximumInterval: Duration::seconds(60),
        nonRetryableExceptions: [\DomainException::class],
        timeouts: ActivityTimeouts::none()->withScheduleToClose(Duration::minutes(10)),
        taskQueue: 'slow-things',
        summary: 'charge the card',
    ),
);
```

`RetryLimit::ofAttempts(3)` means three attempts in total. `RetryLimit::once()` means no
retry at all; `RetryLimit::unlimited()` is the default and matches Temporal's.

## The four timeouts, and which one you actually want

| Timeout | Bounds |
|---|---|
| `scheduleToStart` | how long the task may wait in the queue before a worker takes it |
| `startToClose` | **one attempt** |
| `scheduleToClose` | the whole thing, retries included |
| `heartbeat` | how long a long activity may go silent |

Reach for `startToClose` for "this call should never take more than X", and
`scheduleToClose` for "give up on this step after X whatever happens". A bare `Duration`
passed as `timeouts:` is read as `startToClose`.

## Non-retryable failures

```php
nonRetryableExceptions: [\DomainException::class]
```

A card refused is not a transient failure: retrying it three times annoys the bank and
delays the answer. Declare the exception types that mean *stop*, and let everything else
retry.

## Retries need a worker

On a Temporal backend the retry policy is enforced by the **cluster**: attempts are
scheduled whether or not an activity worker is listening. An execution whose activity
"failed after 3 attempts" within seconds is the signature of a worker that was not
running — not of code that is wrong three times over.

## Writing an activity that survives a retry

The engine records the *result* once. It cannot un-charge a card. So:

- make the side effect idempotent when you can (a business key, an upsert, a provider's
  idempotency key);
- pass the execution id or a business id so the downstream system can deduplicate;
- do the irreversible thing **last** in the activity, after everything that can fail.

## Heartbeats and cancellation

An activity that takes minutes should heartbeat, both to prove it is alive and to learn
that it has been cancelled. With `ActivityCancellationType::TryCancel` (the default) the
workflow moves on as soon as cancellation is requested; the activity finds out at its
next heartbeat.
