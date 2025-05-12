# Starknet Perpetuals: Perpetuals DEX - Findings Report

## Table of contents
- ### High Risk Findings
    - #### H-01. Double subtraction of collateral diff in transfer flow causes incorrect health validation

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://code4rena.com/audits/2025-03-starknet-perpetual)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Double subtraction of collateral diff in transfer flow causes incorrect health validation
## Finding description and impact
In the transfer flow, the collateral diff for the sender’s position is applied immediately via _execute_transfer and then re-applied during the health validation check in _validate_healthy_or_healthier_position through the enrichment process.

This double subtraction results in an artificially low “after” collateral value when computing the position’s health.

Consequently, a legitimate transfer that would leave the position healthy may be erroneously rejected as undercollateralized while being perfectly fine.

This issue directly affects core protocol functionality by potentially preventing valid collateral transfers and may lead to user frustration and loss of trust in the protocol.

---

## Proof of Concept
Let's describe the bug with a simple scenario, and then we'll go over the code to confirm it.

Assume a sender’s position initially has 1000 units of collateral, and his minimum collateral balance for this position to be healthy is 700 units.  
The sender initiates a transfer of 200 units to another position which should execute without a problem.  
The _execute_transfer function subtracts 200 from the sender’s stored collateral, updating it from 1000 to 800.  
Later, during the health validation, _validate_healthy_or_healthier_position is invoked.  
This function calls enrich_position_diff, which fetches the current effective collateral via get_collateral_provisional_balance (which now returns 800).

It then adds the same collateral diff (–200) again to simulate the “after” state, resulting in an effective collateral of 600.

The transfer is rejected because his simulated collateral balance after the transfer is 600 < 700 while in reality he will have 800 units of collateral.  
The function simulates a double spend and as a result will reject legitimate transfers.

Let's go over the transfer flow code. transfer calls _execute_transfer:

```rust
fn _execute_transfer(
    ref self: ContractState,
    recipient: PositionId,
    position_id: PositionId,
    collateral_id: AssetId,
    amount: u64,
) {
    // Parameters
    let position_diff_sender = PositionDiff {
        collateral_diff: -amount.into(), synthetic_diff: Option::None,
    };

    let position_diff_recipient = PositionDiff {
        collateral_diff: amount.into(), synthetic_diff: Option::None,
    };

    // Execute transfer
    self.positions.apply_diff(:position_id, position_diff: position_diff_sender); // <--- called 1st

    self
        .positions
        .apply_diff(position_id: recipient, position_diff: position_diff_recipient); 

    /// Validations - Fundamentals:
    let position = self.positions.get_position_snapshot(:position_id);

    self
        ._validate_healthy_or_healthier_position( // <--- called 2nd
            :position_id, :position, position_diff: position_diff_sender,
        );
}
```

We're going to focus on the fact that `.positions.apply_diff` is called 1st and then `_validate_healthy_or_healthier_position` is called 2nd.

Let's follow what `apply_diff` does:

```rust
fn apply_diff(
    ref self: ComponentState<TContractState>,
    position_id: PositionId,
    position_diff: PositionDiff,
) {
    let position_mut = self.get_position_mut(:position_id);

    position_mut.collateral_balance.add_and_write(position_diff.collateral_diff); // <--- updates the collateral balance with the diff

    if let Option::Some((synthetic_id, synthetic_diff)) = position_diff.synthetic_diff {
        self
            ._update_synthetic_balance_and_funding(
                position: position_mut, :synthetic_id, :synthetic_diff,
            );
    };
}
```

As you can see the collateral is updated in this line:

```
position_mut.collateral_balance.add_and_write(position_diff.collateral_diff);
```

So in our scenario, the collateral becomes from 1000 to 800 here.

Then, in `_validate_healthy_or_healthier_position`:

```rust
fn _validate_healthy_or_healthier_position(
    self: @ContractState,
    position_id: PositionId,
    position: StoragePath<Position>,
    position_diff: PositionDiff,
) {
    let unchanged_synthetics = self
        .positions
        .get_position_unchanged_synthetics(:position, :position_diff);

    let position_diff_enriched = self.enrich_position_diff(:position, :position_diff);

    let tvtr = calculate_position_tvtr_change(
        :unchanged_synthetics, :position_diff_enriched,
    );
    assert_healthy_or_healthier(:position_id, :tvtr);
}
```

We call `enrich_position_diff`:

```rust
fn enrich_position_diff(
    self: @ContractState, position: StoragePath<Position>, position_diff: PositionDiff,
) -> PositionDiffEnriched {
    // unrelated functionality related to synthethic exposures

    let before = self.positions.get_collateral_provisional_balance(:position); // 800

    let after = before + position_diff.collateral_diff; // 800 - 200 = 600

    let collateral_enriched = BalanceDiff { before: before, after };

    PositionDiffEnriched { collateral_enriched, synthetic_enriched }
}
```

Here, `before` will be 800 as the collateral is already updated, and `after` will be 600. And we pass the position to:

```rust
pub fn assert_healthy_or_healthier(position_id: PositionId, tvtr: TVTRChange) {
    let position_state_after_change = get_position_state(position_tvtr: tvtr.after);
    if position_state_after_change == PositionState::Healthy {
        // If the position is healthy we can return.
        return;
    }
}
```

As you can see the `assert_healthy_or_healthier` function gets the `after` state and evaluates if it's healthy. In our scenario, this will fail because `after` is wrongly calculated as 600 while our required margin is 700.

**Impact:** legitimate transfers will get wrongly rejected, breaking core protocol functionality.

---

## Recommended mitigation steps

Modify the transfer flow so that health validation is performed before applying the diff to the stored collateral.

This ensures that the diff is only simulated once during the health check and is consistent with how the rest of the function operate - validate first and then apply the diff.
