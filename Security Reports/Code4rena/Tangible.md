# What is Tangible?
Tangible converts real world assets into tangible NFTs (TNFTs) that can be redeemed for the physical product at any time.

The scope of this audit was CAVIAR - a liquid wrapper from Tangible, removes the complexity and commitment of ve(3,3), creating a simple token for nearly any level of crypto investor. Weekly voting, locking, token management and everything else have been fully automated leaving users with single, simple, high-yield token to stake.


[Link to the contest](https://code4rena.com/contests/2023-08-tangible-caviar)

# Issues found by me

| Severity | Title                                                                                        | Link                                                                       |
| :------- | :------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------- |
| High     | CaviarCompounder.sol#deposit - depositor can mint free shares allowing him to drain the whole contract | [Link]() |
| Medium     | CaviarFeeManager.sol#_swapToUSDR - no slippage protection leaves the function vulnerable to sandwich attacks | [Link]() |
| Medium     | CaviarFeeManager.sol#_swapToUSDR - using block.timestamp as expiration deadline offers no protection | [Link]() |

# 1. CaviarCompounder.sol#deposit - depositor can mint free shares allowing him to drain the whole contract

## Impact
The intended behavior of this function should be that a user deposits CVR in CaviarCompounder, receives shares in return, lets CaviarCompounder do its auto-compounding then the user is able to claim his rewards.

However, in the current implementation of the `deposit()` function, the user doesn't transfer anything to the contract. The function just mints him shares based on the `amount` he provided to the function, which doesn't have any restrictions.

An attacker could easily pass `amount` that results in minting enough shares for him to be able to withdraw all the available CVR in the contract with the `withdraw` function.

## Proof of Concept
https://github.com/code-423n4/2023-08-tangible/blob/d4838f6028f4be80711aaa1d1dc45a8b7fd68b02/contracts/CaviarCompounder.sol#L50

## Tools Used
Manual review
## Recommended Mitigation Steps
Actually transfer the `amount` of CVR from the user to the contract.

# 2. CaviarFeeManager.sol#_swapToUSDR - no slippage protection leaves the function vulnerable to sandwich attacks

## Impact
In CaviarFeeManager.sol, the function _swapToUSDR() doesn't have slippage protection making it vulnerable to sandwich attacks.
`amountOutMin` represents the minimum amount of tokens we are ready to receive and given that is is hardcoded to 0 and there is no other slippage protection, means that probably the contract will get exploited via a sandwich attack while trying to swap.

These attacks are extremely common, and many MEV bots are looking exactly for this kind of unsafe swaps, making the chance of getting sandwiched extremely high.

The impact is loss of funds from sandwich attack when swapping tokens because of lack of slippage control.

## Proof of Concept
https://github.com/code-423n4/2023-08-tangible/blob/d4838f6028f4be80711aaa1d1dc45a8b7fd68b02/contracts/CaviarFeeManager.sol#L111
`amountOutMin` is hardcoded to 0.

## Tools Used
Manual review
## Recommended Mitigation Steps
Calculate the amountOutMinimum earlier in the function or pass it as a parameter. Then check if the contract received the required tokens.

# 3. CaviarFeeManager.sol#_swapToUSDR - using block.timestamp as expiration deadline offers no protection

## Impact
AMMs like Uniswap V3 allow users to specify a deadline parameter that enforces a time limit by which the transaction must be executed. Without a deadline parameter, the transaction may sit in the mempool and be executed at a much later time potentially resulting in a worse price for the user.

However, using `block.timestamp` for the deadline offers no protection at all.

Whenever the miner decides to include the txn in a block, it will be valid at that time, since `block.timestamp` will be the current timestamp(i.e the timestamp of including the txn in the block, doesn't matter when it was submitted).

Therefore, `block.timestamp` offers no protection at all against executing the transaction at a much later time than intended.

A miner can just hold the transaction until maximum slippage is incurred or until he finds a way to benefit from it. The intended functionality of a transaction deadline does not work.

## Proof of Concept
https://github.com/code-423n4/2023-08-tangible/blob/d4838f6028f4be80711aaa1d1dc45a8b7fd68b02/contracts/CaviarFeeManager.sol#L114

## Tools Used
Manual review
## Recommended Mitigation Steps
Add a deadline argument to the function and pass it along to the AMM call.