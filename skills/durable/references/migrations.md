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
  Durable's is `AsActivity::$name . '.' . AsActivityMethod::$name` when the contract name is
  not empty, and the method name alone when it is. `ActivityContractAttributesRector` carries
  over the empty prefix and any literal prefix ending in a dot with something before it:
  `'Order.'` becomes `#[AsActivity(name: 'Order')]`, `'Billing.Order.'` becomes
  `#[AsActivity(name: 'Billing.Order')]`, and for that prefix both engines name the activity
  `Billing.Order.charge`. It changes no attribute and writes a `durable-rector:` marker in two
  cases:
  - the prefix is computed (a constant, a concatenation), does not end in a dot (`'Order'`: the
    SDK type of `charge()` is `Ordercharge`), or is `'.'` alone (the SDK type is `.charge`, and
    an empty contract name gives `charge`): the marker goes above the interface;
  - one method's `#[ActivityMethod(name:)]` is not a string literal: the whole contract stays as
    it is, and the marker goes above that method.

  For each of these markers, choose the Durable names by hand so that the Durable name equals
  the old SDK type exactly. An empty `#[AsActivity(name: '')]` with the full SDK type in
  `#[AsActivityMethod(name:)]` reproduces any prefix: for a prefix `'Order'`, the SDK type of
  `charge()` is `Ordercharge`, and `#[AsActivityMethod(name: 'Ordercharge')]` keeps it.

### 2. SDK attributes left in place

On workflow interfaces, `WorkflowClassAttributesRector` writes `#[AsWorkflow(name:)]`, derived
from `#[WorkflowInterface]`, and copies the method attributes onto the implementing class, where
Durable reads them. It leaves the SDK
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
`LocalActivityOptions`), `Saga` and `Mutex`, and two more kinds of statement:

- a reference to `ApplicationFailure`, `ServerFailure`, `TerminatedFailure` or `TimeoutFailure`
  (in a `catch`, a `new`, a `throw`, an `instanceof`, a static call, a `::class`, a parameter
  type or a return type). Durable has no counterpart for these four, and once `temporal/sdk` is
  removed the reference no longer resolves. PHP does not autoload the class named in a `catch`,
  so such a `catch` never matches and raises no error. A `catch` is marked above its `try`, a
  parameter or return type above its method or function; the `use` import is not marked. Every
  one of these references carries the same text:
  `<Class> has no Durable counterpart — once temporal/sdk is removed this reference no longer resolves; decide by hand`.
  A statement that already carries a `durable-rector:` comment gets no second one, so a marker
  written by an earlier version of the rule keeps its old text
  (`<Class> has no Durable counterpart — a catch on it never matches after migration; decide by hand`).
  The other three SDK failures (`ActivityFailure`, `ChildWorkflowFailure`, `CanceledFailure`) are
  renamed to their Durable counterparts;
- a `Promise::` call that the execution-model rule does not rewrite: any method other than `all`,
  `any` and `some`, any of those three called with no argument, and `some()` called without a
  count.

```bash
grep -rn 'durable-rector:' src/
```

Read the markers before rewriting anything. A workflow built on `Workflow::async()` or
`Workflow::runLocked()` is a redesign, and `Workflow::getVersion()` has no target until workflow
versioning lands. Say so to whoever asked for the migration before starting the rewrite.

`TemporalFacadeToEnvironmentRector` writes the same marker on a **static** workflow method: it
has no `$this`, so it has no environment to call.

### 4. Return types removed, never written

A de-yielded method cannot keep a generator return type (`\Generator`, `Traversable`,
`iterable`), so the rule removes it, and removes it from the
interface too. Nothing replaces it: what the method returns was never declared in the SDK code.
Write each return type by hand; the contract's docblock usually says what it is.

### 5. `Workflow::await()` with more than one condition

The SDK's `await(...$conditions)` is variadic and settles on the first condition. Durable's
`await($condition, $deadline)` takes one condition and a deadline, so a mechanical rewrite would
turn the second condition into a timeout. The rule rewrites `await($c)` and
`awaitWithTimeout($t, $c)`, leaves every other arity as it is, and marks it
(`more than one condition ... combine them by hand`). Combine the conditions into one closure.

## What the migration leaves unchanged without a marker

These carry no `durable-rector:` comment after a run. Check them by hand:

- **A plain iterator generator inside a workflow class.** In a class that implements an SDK
  `#[WorkflowInterface]` contract or calls the facade, every non-static method is de-yielded,
  helpers included, so a method that yields ordinary values is rewritten with the rest. Read
  every method in a migrated workflow class that had a `yield` not followed by a facade call or
  a stub.
- **A generator return type** (`\Generator`, `Traversable`, `iterable`), removed with nothing recording that it was there (case 4).
- **A callable that is not a `\Closure`.** `Workflow::sideEffect([$this, 'compute'])` is
  rewritten as is; Durable takes a `\Closure`, so it throws a `TypeError` on first run. Grep for
  array and string callables before running the workflow.
- **A workflow name the rule did not write.** `WorkflowClassAttributesRector` reads the SDK
  attribute only on the interfaces a class implements: an SDK workflow attribute on the class
  itself gets no `#[AsWorkflow]`. The rule also leaves a class untouched when Rector's reflection
  cannot load it. Without `#[AsWorkflow(name:)]`, Durable's
  workflow type is the class's short name (case 1).
- **The `Temporal\Activity` facade** called from activity code (`Activity::getInfo()`,
  `Activity::heartbeat()`).
- **The client side**: code that starts, signals or queries a workflow through the SDK client.
- **An interceptor**: no rule matches it.

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
3. What it left unchanged without a marker: every item of the section above that applies,
   and the `UPGRADE.md` sections between the two versions.
