# Nexus operations — calling across a boundary

A Nexus operation is the one place in a workflow where **the wait is served by somebody
else**: another team, another namespace, another deployment. That is why it is not an
activity: an activity is your code, a Nexus operation is a contract with someone whose
release calendar is not yours.

## The caller's side

```php
$billing = $this->environment->nexusStub(BillingContract::class, endpoint: 'billing-endpoint');
$check   = $this->environment->await($billing->check($orderId, 4990, 'EUR'));
```

Or without a contract class, when you only have the names:

```php
$this->environment->await($this->environment->nexusOperation(
    endpoint: 'billing-endpoint',
    service: 'billing',
    operation: 'check',
    payload: ['order' => $orderId, 'amountInCents' => 4990, 'currency' => 'EUR'],
));
```

Prefer the stub: the contract gives you the operation names and the argument order, and
it is what makes a rename visible at compile time on the caller's side.

## The server's side

An operation answered **immediately** is a handler method:

```php
#[AsNexusServiceHandler(BillingContract::class)]
final class BillingHandler
{
    public function check(string $order, int $amountInCents, string $currency): array
    {
        return ['accepted' => true, 'reason' => null];
    }
}
```

An operation answered by **a workflow** — because answering takes minutes, or survives a
restart — declares which operation it fulfils, and the server delivers that workflow's
result to the caller:

```php
#[AsWorkflow('billing.charge')]
#[FulfilsNexusOperation(contract: BillingContract::class, operation: 'charge')]
final class ChargeWorkflow { /* … */ }
```

## ⚠ The contract splits in two, and PHP is the reason

A contract holding both kinds cannot be implemented by the handler class: PHP has no way
to say "implements partially", and the operation fulfilled by a workflow has no body
here. So the contract is written as two interfaces:

```php
#[AsNexusService('billing')]
interface BillingServed                    // what the handler implements
{
    #[AsNexusOperation('check')]
    public function check(string $order, int $amountInCents, string $currency): array;
}

#[AsNexusService('billing')]
interface BillingContract extends BillingServed   // what the caller sees
{
    #[AsNexusOperation('charge')]                  // fulfilled by a workflow
    public function charge(string $order, int $amountInCents, string $currency): array;
}
```

The caller types against `BillingContract`; the handler implements `BillingServed`. Both
carry `#[AsNexusService]` with the **same** service name — that name is what travels.

## ⚠ The trap that fails silently: parameter names are the wire format

The payload is keyed **by parameter name**. A parameter renamed on one side and not the
other hands the workflow `null` — no error, no trace, no failed call. Treat the declared
names as the interface they are, and rename them on both sides in the same commit, or
not at all.

## Money, and anything else that must survive a round trip

Pass amounts in the smallest unit, as integers. A float that crosses a JSON encode and
decode is no longer quite the same number, and the operation that reconciles it belongs
to a team that will not enjoy finding out.

## When the backend cannot route it

Nexus — calling **and** serving — is a Temporal capability. The in-memory and SQL
backends have no equivalent, and the library says so explicitly rather than ignoring the
call: `nexusOperation()` throws `NexusUnsupportedByBackendException` at the call site,
and a handler fails when it is registered. Nothing silently does nothing.
