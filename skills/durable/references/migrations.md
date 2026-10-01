# Migrations: Rector first, then the part Rector leaves to you

`gplanchat/durable-rector` carries two sets:

| Set | For |
|---|---|
| `temporal-sdk.php` | Coming off the official Temporal PHP SDK, once |
| `durable-upgrade.php` | Moving from one Durable version to the next, on every upgrade; cumulative |

```bash
composer require --dev gplanchat/durable-rector
```

```php
// rector.php
return Rector\Config\RectorConfig::configure()
    ->withImportNames()   // without it, rewritten names land fully qualified next to a stale `use`
    ->withSets([__DIR__ . '/vendor/gplanchat/durable-rector/config/sets/temporal-sdk.php']);
```

The repository's order for any break is Rector first, a script when Rector cannot do it, and
documentation in every case. Run the set, read the diff, then work through the five cases
below. A migration reported as "Rector ran, done" is not finished until each of them is checked.

## Coming off the Temporal SDK: the five cases

### 1. Workflow and activity type names

Both engines derive a type name, and they derive it differently. A run already started on the
server resolves its code by that name: if the name changes, the code compiles, the tests pass,
and every run in flight stops resolving. No error points at the rename.

- **Workflows.** The SDK uses `#[WorkflowMethod(name:)]`, falling back to the interface's short
  name; Durable uses `#[AsWorkflow(name:)]`, falling back to the class's short name.
  `WorkflowClassAttributesRector` always writes the name out. Over `temporalio/samples-php`, 24
  of the 27 names it writes are ones the fallback would have got wrong. Check that every
  migrated workflow class carries an explicit `#[AsWorkflow(name:)]`. The rule leaves a class
  Rector's reflection cannot load untouched, and keeps the name of a class that already has
  `#[AsWorkflow]`, both with no marker.
- **Activities.** The SDK's type is `prefix . (name ?? methodName)`, with no separator.
  Durable's is `AsActivity::$name . '.' . AsActivityMethod::$name`, and the dot is always
  inserted. The two agree only on an empty prefix and on a single segment ending in a dot
  (`'Order.'`). `ActivityContractAttributesRector` leaves the whole contract untouched, **with no
  marker**, when the prefix:
  - is computed (a constant, a concatenation);
  - does not end in a dot (`'Order'`);
  - contains another dot (`'Billing.Order.'`);
  - or when one method's `#[ActivityMethod(name:)]` is not a string literal.

  Find these afterwards with `grep -rn 'ActivityInterface' src/`: every hit still on an SDK
  attribute is a contract that was not migrated. Choose the Durable names by hand so that
  `contract.method` equals the old SDK type exactly.

### 2. SDK attributes left in place

On workflow interfaces, `WorkflowClassAttributesRector` copies `#[WorkflowInterface]` and the
method attributes onto the implementing class, where Durable reads them, and leaves the SDK
attributes on the interface. A rule cannot read an attribute another rule deleted in the same
pass. Durable ignores them, so they cost nothing at run time. Remove them by hand once the migration
runs; `composer remove temporal/sdk` is the step that forces that cleanup.

### 3. `durable-rector:` markers

`UnmigratableTemporalCallRector` writes a `// durable-rector: <reason>` comment above every
statement calling something Durable has no counterpart for, and changes nothing else. It works
from an allow-list: seven facade methods (`newActivityStub`, `newChildWorkflowStub`, `await`,
`awaitWithTimeout`, `timer`, `sideEffect`, `continueAsNew`) are rewritten by the execution-model
rule; every other `Workflow::` call is marked. It also marks the SDK options objects
(`ActivityOptions`, `RetryOptions`, `ChildWorkflowOptions`, `ContinueAsNewOptions`,
`LocalActivityOptions`), `Saga` and `Mutex`.

```bash
grep -rn 'durable-rector:' src/
```

Read the markers before rewriting anything. A workflow built on `Workflow::async()` or
`Workflow::runLocked()` is a redesign, and `Workflow::getVersion()` has no target until workflow
versioning lands. Say so to whoever asked for the migration before starting the rewrite.

`TemporalFacadeToEnvironmentRector` writes the same marker on a **static** workflow method: it
has no `$this`, so it has no environment to call.

### 4. Return types removed, never written

A de-yielded method cannot keep `\Generator`, so the rule removes it, and removes it from the
interface too. Nothing replaces it: what the method returns was never declared in the SDK code.
Write each return type by hand; the contract's docblock usually says what it is.

### 5. `Workflow::await()` with more than one condition

The SDK's `await(...$conditions)` is variadic and settles on the first condition. Durable's
`await($condition, $deadline)` takes one condition and a deadline, so a mechanical rewrite would
turn the second condition into a timeout. The rule rewrites `await($c)` and
`awaitWithTimeout($t, $c)`, leaves every other arity as it is, and marks it
(`more than one condition ... combine them by hand`). Combine the conditions into one closure.

## What no rule detects

- **A plain iterator generator inside a workflow class.** In a class that implements an SDK
  `#[WorkflowInterface]` contract or calls the facade, every non-static method is de-yielded,
  helpers included. A method that yields ordinary values is rewritten with the rest. Read every
  method in a migrated workflow class that had a `yield` not followed by a facade call or a stub.
- **SDK failures with no rename.** The set renames `ActivityFailure`, `ChildWorkflowFailure` and
  `CanceledFailure`. `ApplicationFailure`, `ServerFailure`, `TerminatedFailure` and
  `TimeoutFailure` are left as they are. PHP does not autoload the class named in a `catch`, so
  after `composer remove temporal/sdk` such a block no longer matches anything, and raises no
  error. `grep -rn 'Temporal\\Exception' src/` finds them.
- **A callable that is not a `\Closure`.** `Workflow::sideEffect([$this, 'compute'])` is
  rewritten as is; Durable takes a `\Closure`, so it throws a `TypeError` on first run. Grep for
  array and string callables before running the workflow.

## Moving between Durable versions

`durable-upgrade.php` renames classes and rewrites the calls it can prove. What it cannot do is
written per version in `UPGRADE.md` at the root of `gplanchat/durable-dev`: read every section
between your old version and the new one. One case recurs: when a class moves between packages,
a Symfony project's compiled container keeps the old name. Run `bin/console cache:clear` after
the upgrade, or the failure lands on the first call after deployment.

## What to report

Hand back three lists:

1. What Rector rewrote (the diff).
2. What it marked: the output of `grep -rn 'durable-rector:' src/`, with a decision for each.
3. What it could not see: the leftover `ActivityInterface` contracts from case 1, the return
   types from case 4, the plain generators, the unrenamed `catch` blocks, the non-`\Closure`
   callables, and the `UPGRADE.md` sections that apply.
