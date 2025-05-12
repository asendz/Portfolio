# THORWallet: All-in-one DeFi solution - Findings Report

## Table of contents
- ### High Risk Findings
    - #### H-01. 
- ### Medium Risk Findings
    - #### M-01. 

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://code4rena.com/audits/2025-02-thorwallet)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Uncapped TGT deposits leading to oversubscription of claimable TITN
## Finding description and impact

The `MergeTgt` contract uses the formula:

```solidity
titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE
```

where `TGT_TO_EXCHANGE = 579,000,000 * 10^18` and `TITN_ARB = 173,700,000 * 10^18`. However, there is no mechanism to cap the cumulative TGT deposited. If users deposit more TGT than `TGT_TO_EXCHANGE`, the total claimable TITN can exceed `TITN_ARB`. This oversubscription can result in:

- **Insufficient TITN for Claims**: Users may not be able to claim their full allocation.
- **Financial Loss**: Users that are late to claim won't receive their TITN since there's no enough TITN balance.

---

## Proof of Concept

Scenario:  
Assume `TGT_TO_EXCHANGE = 579,000,000` and `TITN_ARB = 173,700,000`. If users deposit a total of 600M TGT, the computed claimable TITN becomes:

```text
claimableTITN = (600M * 173.7M) / 579M = ~180M TITN
```

This exceeds the `TITN_ARB` allocation by ~6.3M TITN. Consequently, when users attempt to claim via `claimTitn` or later trigger `withdrawRemainingTitn`, there won’t be sufficient TITN to cover all claims.

# Medium Risk Findings
## <a id='M-01'></a>M-01. Persistent `isBridgedTokenHolder` status restricting non-bridged token transfers on Base
## Finding description and impact

The `Titn` contract facilitates token bridging from Arbitrum to Base.

When a user receives bridged tokens via the `_credit` function, their address is permanently marked as a bridged token holder by setting `isBridgedTokenHolder[_to] = true`.

This flag remains true indefinitely, as there is no mechanism in the contract to unset it, even if the user transfers all bridged tokens away **OR** had non-bridged tokens beforehand.

When `isBridgedTokensTransferLocked` is true, bridged token holders on BASE are restricted to transferring tokens only to the `transferAllowedContract` (e.g., a staking contract) or the LayerZero endpoint (`lzEndpoint`).

The vulnerability arises because this persistent status can unfairly restrict an address’s ability to transfer **non-bridged** TITN tokens that are acquired by another method.

If a user bridges tokens from Arbitrum to Base, transfers all bridged tokens away, and subsequently receives non-bridged TITN tokens, they remain classified as a bridged token holder.

This subjects their entire token balance—now consisting solely of non-bridged tokens—to transfer restrictions, contradicting the documentation’s invariant that restrictions apply only to holders of bridged tokens.

---

## Proof of Concept

**Referenced code of the `_credit` function:**

```solidity
function _credit(
    address _to,
    uint256 _amountLD,
    uint32 /*_srcEid*/
) internal virtual override returns (uint256 amountReceivedLD) {
    if (_to == address(0x0)) 
        _to = address(0xdead); // _mint(...) does not support address(0x0)
    _mint(_to, _amountLD);
    if (!isBridgedTokenHolder[_to]) {
        isBridgedTokenHolder[_to] = true;
    }
    return _amountLD;
}
```

The issue stems from the persistent `isBridgedTokenHolder` status, which remains even after a user transfers all bridged tokens away without taking into account the fact that the user could've had non-bridged tokens beforehand, or received non-bridge tokens after that.

This allows us to demonstrate how a user’s non-bridged tokens become restricted, violating the invariant that restrictions apply only to bridged token holders. Here’s the scenario:

1. User A had 10 TINT on Base  
2. User A received 10 TINT bridged from Arbitrum  
3. The flag is set: `isBridgedTokenHolder[A] = true`  
4. **Result**: The user's entire balance is treated like bridged tokens even though he had non-bridged tokens before that.  

This persists indefinitely unless the contract logic changes, affecting any user who ever received bridged tokens.

The documentation states that:

> "Non-bridged TITN Tokens: Holders can transfer their TITN tokens freely to any address as long as the tokens have not been bridged from ARBITRUM."

This persists indefinitely even if the user transfers all of his tokens and receives non-bridged tokens afterward.

---

## Recommended mitigation steps

Track bridged token balances. Introduce a mapping to track bridged token balances and impose restrictions only on that part of the user's balance. Additionally, reset the flag when the bridged-token balance reaches zero.