# Workflows — beyond the happy path

## Signals and updates

A **signal** is fire-and-forget from the outside; an **update** expects an answer.
Both are declared on a contract and handled inside the workflow:

```php
interface CheckoutSignals
{
    #[AsSignalMethod('approved')]        // a BackedEnum works too
    public function approved(string $by): void;

    #[AsUpdateMethod('address')]
    public function changeAddress(string $address): bool;
}
```

```php
#[AsWorkflowMethod]
public function run(string $orderId): string
{
    $approved = false;
    $this->environment->onSignal('approved', function (string $by) use (&$approved): void {
        $approved = true;
    });

    // Wait for a condition, with a deadline. Without one, this waits forever — which is
    // legitimate for a workflow, and a bug if you meant "give up after a while".
    $this->environment->await(fn(): bool => $approved, Duration::hours(48));
}
```

`onUpdate()` is the same shape, and its handler's return value goes back to the caller.

## Child workflows

```php
$child = $this->environment->childWorkflowStub(ShipmentWorkflow::class);
$result = $this->environment->await($child->run($orderId));
```

A child has its own execution, its own journal and its own line in the observation
timeline. Use one when the sub-process has a life of its own — its own retries, its own
cancellation, its own history worth reading. Use an activity when it is just work.

## Timers

```php
$this->environment->sleep(Duration::seconds(30), 'before retrying the gateway');

$timer = $this->environment->timer(Duration::minutes(5), 'payment window');
$done  = $this->environment->any($timer, $paymentAwaitable);
```

**Always pass the summary.** It is the timer's only business fact, and it is what the
admin timeline shows as the row's name — without it an operator reads `timer 5.0 s` and
learns the duration but not the reason.

## Side effects and versioning

```php
$id = $this->environment->sideEffect(fn(): string => Uuid::v7()->toRfc4122());
```

Journaled once: the replay reads the recorded value instead of drawing a new one.

```php
$v = $this->environment->version('add-fraud-check', minSupported: 1, maxSupported: 2);
if ($v >= 2) {
    $this->environment->await($orders->fraudCheck($orderId));
}
```

Executions started before the change keep answering `1` and skip the branch; new ones
answer `2`. This is how you deploy a change without breaking runs already in flight.

## continueAsNew

A workflow that loops forever grows a journal forever. Cut it:

```php
if ($processed > 1000) {
    $this->environment->continueAsNew(self::class, ['cursor' => $cursor]);
}
```

It never returns: the current execution ends and a fresh one starts with the payload,
same workflow id, empty history.

## Starting an execution

⚠ **Never run a workflow inline in a web request.** The request ends, the process is
recycled, and the execution dies with it — which is the failure durable execution exists
to remove.

```php
$factory->workflowClient()->startAsync(
    CheckoutWorkflow::class,
    ['orderId' => $order->getIncrementId()],
    'order-' . $order->getIncrementId(),   // the execution id: yours, and idempotent
);
```

Give the execution a **business** id. Starting twice with the same id does not start two
executions, which is exactly what you want from a controller that can be double-posted.

And an observer that starts a workflow must not throw: a sale that already happened
stays happened. A workflow that fails to start is an operational incident, not a reason
to refuse the customer — refusing them would not give the money back either.

## Declaring the workflow to the host

| Host | How |
|---|---|
| Symfony, Sylius | autoconfigured from the attribute |
| Laravel | **declared, not scanned** — the `workflows` key of `config/durable.php` names the classes |
| Magento | listed explicitly in `di.xml` — the container has no tag autoconfiguration |
| No framework | you register it on the runtime yourself |

Only Symfony has attribute autoconfiguration. On the other two the class is named
somewhere, and a workflow that runs in a test but not in production is almost always a
class nobody declared.
