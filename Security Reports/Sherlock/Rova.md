# Rova: Onchain launchpad - Findings Report

## Table of contents
- ### Medium Risk Findings
    - #### M-01. Mixing currency and token units in updateParticipation causes wrong accounting and potential loss of funds

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://audits.sherlock.xyz/contests/498)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1

# Meidum Risk Findings

## <a id='M-01'></a>M-01. Mixing currency and token units in updateParticipation causes wrong accounting and potential loss of funds
# Summary

The `updateParticipation` function uses the difference in payment currency amounts to adjust a user’s token allocation instead of using the difference in token amounts. As a result, when a user updates their participation to reduce their commitment, the decrease in recorded tokens is far less than intended—allowing the user to effectively gain extra tokens for the same payment and leading to big errors in accounting.

---

## Example Scenario:

### Initial Participation:
- User participates for 100 tokens at a price of 0.05 per token, paying 5.00 currency units.
- The user’s participation is recorded as 100 tokens.

### Intended Update:
- The user decides to reduce their participation to 80 tokens.
- **Correct calculation**: New payment should be 80 × 0.05 = 4.00 units; thus, the refund should be 5.00 – 4.00 = 1.00 unit, and the new recorded token allocation should be 80 tokens.

### Buggy Behavior (Current implementation):
- The contract calculates the refund as 1.00 currency unit, but then subtracts this 1.00 (interpreted as 1 token unit) from the recorded 100 tokens, resulting in a new allocation of 99 tokens instead of 80 tokens.

---

## Impact:

- **For the User**: The user effectively receives 19 extra tokens (99 tokens instead of 80) for the same payment, reducing their effective cost per token.
- **For the Protocol**: Mis-tracking token allocations may allow users to bypass maximum allocation limits and can lead to oversubscription and financial imbalances in the token sale.

---

## Root Cause

**Reference to the code**:  
https://github.com/sherlock-audit/2025-02-rova/blob/main/rova-contracts/src/Launch.sol#L351-L374

In `updateParticipation`:

```solidity
if (prevInfo.currencyAmount > newCurrencyAmount) {
    
    uint256 refundCurrencyAmount = prevInfo.currencyAmount - newCurrencyAmount;

    if (userTokenAmount - refundCurrencyAmount < settings.minTokenAmountPerUser) {  // <--- mixing sale tokens with currency amount
        revert MinUserTokenAllocationNotReached(
            request.launchGroupId, request.userId, userTokenAmount, request.tokenAmount
        );
    }

    userTokens.set(request.userId, userTokenAmount - refundCurrencyAmount); // <--- mixing sale tokens with currency amount

    IERC20(request.currency).safeTransfer(msg.sender, refundCurrencyAmount);

} else if (newCurrencyAmount > prevInfo.currencyAmount) {

    uint256 additionalCurrencyAmount = newCurrencyAmount - prevInfo.currencyAmount;

    if (userTokenAmount + additionalCurrencyAmount > settings.maxTokenAmountPerUser) { // <--- mixing sale tokens with currency amount
        revert MaxUserTokenAllocationReached(
            request.launchGroupId, request.userId, userTokenAmount, request.tokenAmount
        );
    }

    userTokens.set(request.userId, userTokenAmount + additionalCurrencyAmount); // <--- mixing sale tokens with currency amount

    IERC20(request.currency).safeTransferFrom(msg.sender, address(this), additionalCurrencyAmount);
}
```

---

## Attack Path

1. User participates for 100 tokens, paying 5.00 currency units.
2. User submits an `updateParticipation` request to reduce participation to 80 tokens.
3. The contract calculates a refund of 1.00 currency unit (i.e., a difference based on currency amounts).
4. Instead of reducing the token allocation by 20 tokens, the contract subtracts 1 token unit.
5. The user’s participation is updated to 99 tokens instead of the intended 80 tokens.

---

## Impact

Wrong accounting, leading to losses for users/protocol depending on if the user updates to decrease or increase his allocation.


## Mitigation

Modify the `updateParticipation` function to compute the token difference directly. For example, if reducing participation, subtract the token difference:

```solidity
uint256 tokenDifference = prevInfo.tokenAmount - request.tokenAmount;
userTokens.set(request.userId, userTokenAmount - tokenDifference);
```

And use the currency difference solely to determine the refund amount without affecting the token allocation. This ensures that the user’s recorded token participation accurately reflects their intended update.
