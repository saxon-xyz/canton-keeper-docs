---
layout: default
title: Configuration Reference
---

# Configuration Reference

Jobs are defined in `canton-keeper.yaml`:

```yaml
defaults:
  pollIntervalMs: 10000        # how often to check for actionable contracts
  deduplicationSeconds: 300    # ledger command deduplication window

jobs:
  my-job-name:
    trigger: deadline|match|exists|interval
    watch:
      module: Your.Daml.Module
      entity: YourTemplate
    # ... trigger-specific fields ...
    exercise:
      choice: YourTemplate_SomeChoice
      args:
        fieldName: "$party"
```

## Trigger Types

### `deadline` — Exercise when a timestamp passes

```yaml
cancel-expired:
  trigger: deadline
  watch:
    module: Your.App.Module
    entity: YourTemplate
  when:
    field: validUntil        # field path to timestamp
    condition: past          # fire when now > field
  exercise:
    choice: YourTemplate_Cancel
    args:
      cancelor: "$party"
```

### `match` — Exercise when correlated contracts exist

```yaml
settle-trades:
  trigger: match
  watch:
    module: Your.App.Module
    entity: Trade
  match:
    module: Splice.AmuletAllocation
    entity: AmuletAllocation
    ref: allocation.settlement.settlementRef.cid??   # path on secondary → primary contractId
    leg: allocation.transferLegId                      # path on secondary → leg key
    legs: transferLegs                                 # TextMap field on primary (keys = leg IDs)
  deadline: settleBefore    # optional: skip if past this timestamp
  exercise:
    choice: Trade_Settle
    args:
      allocationsWithContext:
        $eachLeg:
          _1: "$match.contractId"
          _2:
            context:
              values: {}
            meta:
              values: {}
```

### `exists` — Exercise when a contract appears

```yaml
execute-accepted:
  trigger: exists
  watch:
    module: Your.App.Module
    entity: AcceptedAction
  exercise:
    choice: AcceptedAction_Execute
    args: {}
```

### `interval` — Exercise periodically

```yaml
bill-fees:
  trigger: interval
  watch:
    module: Your.App.Module
    entity: Agreement
  every: 86400000    # milliseconds (24 hours)
  exercise:
    choice: Agreement_Bill
    args:
      currentTime: "$now"
```

## Field Paths

Dot-separated paths navigate nested Daml Records:

| Path | What it does |
|------|-------------|
| `validUntil` | Top-level field |
| `amount.initialAmount` | Nested record field |
| `settlementRef.cid??` | Navigate to `cid`, unwrap Optional |
| `allocation.settlement.settlementRef.cid??` | Deep nested with Optional unwrap |

The `??` suffix unwraps Daml `Optional` values — returns the inner value or skips the contract if `None`.

## Choice Argument Expressions

| Expression | Produces |
|---|---|
| `$party` | Venue/operator party |
| `$now` | Current timestamp |
| `$contract.contractId` | Watched contract's ID |
| `$contract.field.path` | Extract field from watched contract |
| `$match.contractId` | Matched secondary contract's ID (match trigger only) |
| `$eachLeg` | Iterate matched legs, build TextMap entry per leg (match trigger only) |
| `{}` | Empty TextMap |
| `"text"` | Text literal |
| `42` | Integer literal |
| `true` / `false` | Boolean literal |
| Nested object | Nested Daml Record |
| Array | Daml List |
