# What is Footium?

Footium is a multiplayer football management game where players own and manage their own digital football club(NFT).
Description
[Link to the contest](https://audits.sherlock.xyz/contests/71)

# Issues found by Asen

| Severity                                                                     | Title                                                  | Link                                                                         |
| :--------------------------------------------------------------------------- | :----------------------------------------------------- | :--------------------------------------------------------------------------- |
| High                                                                         | Club buyers on secondary marketplace could get scammed |
| [Link](https://github.com/sherlock-audit/2023-04-footium-judging/issues/166) |
| Medium                                                                       | Unsafe ERC20.transfer() - unchecked return values      | [Link](https://github.com/sherlock-audit/2023-04-footium-judging/issues/149) |
| Medium                                                                       | Use safeMint instead of mint for ERC721                | [Link](https://github.com/sherlock-audit/2023-04-footium-judging/issues/67)  |

# 1. Club buyers on secondary marketplace could get scammed

## Summary

A seller of a club NFT could scam buyers on NFT marketplaces.

The buyer will expect to get the club NFT + all players in the escrow contract but instead, he could receive just the club NFT and an empty escrow.

## Vulnerability Detail

After talking with the sponsor, he confirmed that the value of a club is based mainly on its division, its history, and its players which are held in the escrow contract that is managed by the ClubOwner.

Let's consider the following scenario:

I own a club with a lot of very valuable players and I list it on a secondary marketplace
A buyer comes in and submits a transaction to buy my club expecting to get all the very valuable players that belong to that club
However, I am monitoring the mem pool so when I see this transaction, I front-run it with my own which transfers all the players out of the escrow contract
Buyer receives an expensive club NFT but all the players are not there already
Buyer is left with an empty club although one of the main reasons he decided to buy it was because of the club's players.

Also, the attacker could continue doing this, luring buyers into buying new clubs based on the valuable players they have. Eventually leading to either a lot of people getting scammed and/or this becomes well-known and nobody will want to buy clubs on the secondary marketplaces. At least not for their players.

## Impact

Buyers on secondary marketplaces will get scammed expecting that they'll receive the valuable players of the club but receive an empty club instead which in turn will damage severely the protocol's reputation.

## Code Snippet

https://github.com/sherlock-audit/2023-04-footium/blob/11736f3f7f7efa88cb99ee98b04b85a46621347c/footium-eth-shareable/contracts/FootiumEscrow.sol#L15

## Recommendation

I thought about a couple of solutions for this issue. The most simple that should mitigate it is:

Implement a locked/unlocked functionality for the Escrow contract which makes it not possible to transfer assets while it is locked.
The club NFT contract should be transferrable ONLY if the escrow contract is locked.
Add a short timelock(a minute will be sufficient for 99% of the cases) for locking/unlocking preventing the seller from quickly locking/unlocking again.
That way, the seller will be required to lock the Escrow and such an attack won't be possible. When the buyer receives his club NFT, he simply unlocks the escrow and can use his assets.

You could implement that functionality by rewriting the ERC721 transfer functions making them set the escrow contract to locked before transferring the club NFT.
Or you could rewrite the ERC721 \_beforeTokenTransfer hook which could set the escrow contract to locked.

# 2. Unsafe ERC20.transfer() - unchecked return values

## Summary

The ERC20.transfer() and ERC20.transferFrom() functions return a boolean value indicating success. This parameter needs to be checked for success.

Some tokens do not revert if the transfer failed but return false instead. (e.g. ZRX)

## Vulnerability Detail

The unsafe .transfer() function is used in the claimERC20Prize function in the FootiumPrizeDistributor.sol contract.

```solidity
    function claimERC20Prize(/*parameters*/)
external whenNotPaused nonReentrant {
     //validations

        uint256 value = _amount - totalERC20Claimed[_token][_to];

        if (value > 0) {
            totalERC20Claimed[_token][_to] += value;
            _token.transfer(_to, value);
        }

        emit ClaimERC20(_token, _to, value);
    }
```

Implemented that way, there's a risk of a silent transfer failure which would lead to the internal balance of totalERC20Claimed being updated as the user supposedly received his tokens while in fact, he didn't receive anything.

This will lead to his tokens being stuck in the contract.

## Impact

Could lead to stuck user funds in the contract, if a reward token that doesn't revert on failure but returns false instead is used.

## Code Snippet

https://github.com/sherlock-audit/2023-04-footium/blob/11736f3f7f7efa88cb99ee98b04b85a46621347c/footium-eth-shareable/contracts/FootiumPrizeDistributor.sol#L106

https://github.com/sherlock-audit/2023-04-footium/blob/11736f3f7f7efa88cb99ee98b04b85a46621347c/footium-eth-shareable/contracts/FootiumEscrow.sol#L110

## Recommendation

Recommend using OpenZeppelin's SafeERC20 versions with the safeTransfer and safeTransferFrom functions that handle the return value check as well as non-standard-compliant tokens.

# 3. Use safeMint instead of mint for ERC721

## Summary

In FootiumClub.sol contract, function safeMint uses the internal OZ \_mint function. Should use \_safeMint instead.

## Vulnerability Detail

A new football club NFT will be minted to the to address in the safeMint function:

```solidity
  function safeMint(address to, uint256 tokenId)
        external
        onlyRole(MINTER_ROLE)
        nonReentrant
        whenNotPaused
    {
        FootiumEscrow escrow = new FootiumEscrow(address(this), tokenId);
        clubToEscrow[tokenId] = escrow;

        _mint(to, tokenId);

        emit EscrowDeployed(tokenId, escrow);
    }
```

However, if to is a contract address that does not support ERC721, the NFT can be frozen in the contract.

As per the documentation of EIP-721:

A wallet/broker/auction application MUST implement the wallet interface if it will accept safe transfers.

Ref: https://eips.ethereum.org/EIPS/eip-721

As per the documentation of ERC721.sol by Openzeppelin the usage of \_mint is discouraged:

Ref: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/a7ee03565b4ee14265f4406f9e38a04e0143656f/contracts/token/ERC721/ERC721.sol#L253

## Impact

Could lead to the loss of NFTs.

## Code Snippet

https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumClub.sol#L56

## Recommendation

Use \_safeMint instead of \_mint to check received address support for ERC721 implementation.
