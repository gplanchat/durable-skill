# durable-skill

A Claude Code skill for writing [Durable](https://durable.rocks) workflows, activities
and Nexus operations in PHP.

It teaches an agent the three shapes — a workflow class, an activity contract and its
handler, a Nexus contract — the determinism rule that binds a replayed workflow body,
and the handful of traps that cost a production incident: an activity name written as a
string, a timer without a summary, a Nexus parameter renamed on one side only, an
execution started inline in a web request.

## Install

```
/plugin marketplace add gplanchat/durable-skill
/plugin install durable@durable-skill
```

That is the whole installation. The skill loads itself when a task mentions a durable
workflow, an activity contract, a Nexus operation, or any of the library's attributes.

To install it by hand instead, copy `skills/durable/` into `~/.claude/skills/durable/`.

## What is inside

| File | What it carries |
|---|---|
| `skills/durable/SKILL.md` | The three shapes, the determinism rule, the checklist |
| `skills/durable/references/workflows.md` | Signals, updates, child workflows, timers, versioning, `continueAsNew`, starting an execution |
| `skills/durable/references/activities.md` | Options, retry policy, the four timeouts, writing an activity that survives a retry |
| `skills/durable/references/nexus.md` | Contracts, why a contract splits in two, the payload trap |

The references are loaded on demand, so a task about activities does not pay for the
Nexus reference.

## Which package you need

The skill is about the code you write, which is the same on every host. What differs is
what you install:

| Your situation | Command |
|---|---|
| Learning, or unit tests only | `composer require gplanchat/durable` |
| Symfony | `composer require gplanchat/durable-bundle` |
| Sylius | `composer require gplanchat/durable-plugin` |
| Laravel, one SQL database | `composer require gplanchat/durable gplanchat/durable-bridge-illuminate` |
| Magento 2.4 / Mage-OS | `composer require gplanchat/durable-magento:dev-main` |

Add `gplanchat/durable-bridge-temporal` for a Temporal cluster, or
`gplanchat/durable-bridge-dbal` for one SQL database.

⚠ **The suite is in alpha, and two packages have no tagged version yet.**
`gplanchat/durable` and the bridges publish `v0.1.0-alpha*`, so a project on the default
`minimum-stability: stable` needs `"minimum-stability": "alpha"` and
`"prefer-stable": true`, or an explicit `:^0.1.0@alpha` on each line.
`gplanchat/durable-laravel` and `gplanchat/durable-magento` are on Packagist with
**`dev-main` only** — their first version comes from the next tag. The canonical table,
kept current, is on [durable.rocks](https://durable.rocks).

## License

MIT. See [LICENSE](LICENSE).
