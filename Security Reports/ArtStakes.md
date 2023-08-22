A time-boxed security review of the **ArtStakes** protocol was done, with a focus on the security aspects of the implementation.

A smart contract security review can never verify a complete absence of vulnerabilities nor it could guarantee finding any of them. The only guarantee is that I've spent a sufficient amount of time reviewing and understanding the protocol and did my best to find security breaches and mistakes.

Commit hash - [62348787bd3099a564099e8f8779d124d389eaf6](https://github.com/owl11/ArtStakes/commit/62348787bd3099a564099e8f8779d124d389eaf6)

# What is ArtStakes?

ArtStakes is a protocol seeking to unlock usecases for NFT's like DAO-fying rare collectibles on chain by locking them up on L1 and receiving corresponding ERC721 or ERC20 tokens on L2. An NFT owner can turn it into a DAO or sell shares of it to people that actually want it.

# Findings table

| Severity | &nbsp; &nbsp;ID&nbsp; &nbsp; |                                                                 Title                                                                 |
| :------: | :--------------------------: | :-----------------------------------------------------------------------------------------------------------------------------------: |
|   High   |            [H-01]            |              Re-entrancy in `mintXNFT` in `L2ArtStakes.sol` letting anyone able to mint a lot of NFTs on the L2 network               |
|   High   |            [H-02]            | Only 75% of the total supply of ERC20 tokens(shares) are needed to claim the original NFT opening a lot of opportunities for scamming |
|   High   |            [H-03]            |                                User can get a free corresponding NFT on the L2 by abusing old metadata                                |
|   Low    |            [L-01]            |                              Pragma set as ^0.8.14 which can lead to problems on Optimism and other L2s                               |
|   Low    |            [L-02]            |                                     Single-step process for critical ownership transfer is risky                                      |
|   Low    |            [L-03]            |                           Hardcoded values for `crossDomainMessengerAddr` and `xorigin` in L1ArtStakes.sol                            |

# Detailed findings

Expected user flow:

User has a NFT on L1 -> Registers it's metadata via `registerMetaData` -> `stakeNFT` by transferring it to L1 contract and message the metaData to L2 via `sendMetaData`, at this point the meta data is registered on the L2 contract as well

-> then user `deployCorrospondingToken` on the L2 -> If he wants to unstake, calls `burnERC721` which burns the L2 ERC721 and sends the message to the L1 via `sendBurnerMetaDataX` -> then he calls BurnerUNStakeNFT on the L1, which transfers his NFT from the contract to him.

# [H-01] Re-entrancy in `mintXNFT` in `L2ArtStakes.sol` letting anyone able to mint a lot of NFTs on the L2 network

## Summary

The current implementation of `mintXNFT` doesn't follow the Checks-Effects-Interactions pattern and calls `safeMint` which is vulnerable to re-entrancy because of the `_beforeTokenTransfer` and `_afterTokenTransfer` hooks implemented in the OpenZeppelin's `safeMint` function.

```javascript
function mintXNFT() public returns (bool) {
        require(hasMetadata[msg.sender], "no metadata registered");

        bytes memory data = userMetadata[msg.sender];
        (
            uint256 _tokenId,
            ,
            ,
            ,
            ,
            string memory _uri,
            ,
            address _NFTAddr //NFT address on L1
        ) = getUserMetadata(data);

        require(
            L1NftCloneDeployed[_NFTAddr] = true,
            "you must deploy l2 Nft First"
        );
        require(!mintedTokens[_NFTAddr][_tokenId], "Token ID already minted");

        ERC721X deployed = L1L2AddrToMapping[_NFTAddr];
        //Interactions
        uint256 L2TokenId = deployed.safeMint(msg.sender, _uri);
        //Effects
        L1ToL2TokenId[_tokenId] = L2TokenId;
        mintedTokens[_NFTAddr][_tokenId] = true;// <-- only here we set flag the token as minted
        return true;
    }
```

Let's also see how the custom ERC721X `safeMint` function is implemented:

```javascript
    function safeMint(
        address to,
        string memory uri
    ) public onlyOwner returns (uint256) {
        uint256 tokenId = _tokenIdCounter.current();
        _tokenIdCounter.increment();
        _safeMint(to, tokenId);
        _setTokenURI(tokenId, uri);
        totalSupply();
        return tokenId;
    }
```

It increments the `tokenCounter` which is the `tokenId` and then calls OZ's `_safeMint`:

```javascript
function _safeMint(address to, uint256 tokenId, bytes memory data) internal virtual {
        _mint(to, tokenId);
        require(
            _checkOnERC721Received(address(0), to, tokenId, data),
            "ERC721: transfer to non ERC721Receiver implementer"
        );
    }
function _mint(address to, uint256 tokenId) internal virtual {
        require(to != address(0), "ERC721: mint to the zero address");
        require(!_exists(tokenId), "ERC721: token already minted");

        _beforeTokenTransfer(address(0), to, tokenId, 1);

        // Check that tokenId was not minted by `_beforeTokenTransfer` hook
        require(!_exists(tokenId), "ERC721: token already minted");

        unchecked {
            // Will not overflow unless all 2**256 token ids are minted to the same owner.
            // Given that tokens are minted one by one, it is impossible in practice that
            // this ever happens. Might change if we allow batch minting.
            // The ERC fails to describe this case.
            _balances[to] += 1;
        }

        _owners[tokenId] = to;

        emit Transfer(address(0), to, tokenId);

        _afterTokenTransfer(address(0), to, tokenId, 1);
    }
```

A malicious user could have created a smart contract that implements `_afterTokenTransfer` to re-enter in the `mintXNFT` function. This is possible because the line which set's if a token is already minted is called only after the call to `safeMint`.

## Impact

A malicious user could stake only 1 NFT on the L1 and mint a lot of NFTs on the L2 opening a lot of possibilities for scamming other users.

## Recommendation

You can add nonReentrant by OpenZeppelin and the CEI pattern should always be followed.

# [H-02] Only 75% of the total supply of ERC20 tokens(shares) are needed to claim the original NFT opening a lot of opportunities for scamming

## Summary

When a user stakes an NFT in the ArtStakes protocol, he can choose to receive ERC20 tokens on the L2, representing shares of the original NFT. However, currently only 75% of the totalSupply of the said ERC20 tokens are needed to claim the original NFT:

```javascript
function _burnMajorityERC20(AS_ERC20 _erc20) internal {
        uint256 balance = _erc20.balanceOf(msg.sender);
        uint256 totalSupply = _erc20.totalSupply();

        uint256 minimumBalance = (totalSupply * 75) / 100;
        require(balance >= minimumBalance, "insufficient balance");

        _erc20.burnFrom(msg.sender, balance);
        _erc20.pause();
    }
```

This opens up a lot of possibilities for scamming as a malicious owner could sell up to 25% of the tokens to unknowing buyers thinking they'll get a share of an expensive NFT, for example. Then he could simply burn his 75% of tokens and claim the NFT leaving the buyers only with worthless tokens.

## Impact

People will definitely get scammed by malicious owners with this.

## Recommendation

I don't see a straight forward fix with the current design. If 100% of the supply is required then this would be vulnerable to one person holding 1 token and not letting the person owning all other tokens claim his NFT. Requiring 99% of the supply kind of solves both issues but not really.

Maybe add a method of selling the original NFT when X% of the ERC20 totalSupply holders request for it, and then reimburse them the rewards.

# [H-03] User can get a free corresponding NFT on the L2 by abusing old metadata

## Summary

When a user stakes his NFT on the L1 contract, it sends a message to the L2 contract and the metadata gets registered:

```javascript
function registerMetadata(
        bytes memory _Data
    ) public onlyApprovedAddressXOrigin {
        bytes memory data = _Data;
        (, , , , , , address _owner, ) = getUserMetadata(data);
        userMetadata[_owner] = data;
        hasMetadata[_owner] = true;
        emit MetadataSet(msg.sender, tx.origin, getXorig(), _owner, data);
    }
```

Which is needed when the user wants to mints his NFT on the L2:

```javascript
function mintXNFT() public returns (bool) {
        require(hasMetadata[msg.sender], "no metadata registered");

        bytes memory data = userMetadata[msg.sender];
        (
            uint256 _tokenId,
            ,
            ,
            ,
            ,
            string memory _uri,
            ,
            address _NFTAddr //NFT address on L1
        ) = getUserMetadata(data);

        require(
            L1NftCloneDeployed[_NFTAddr] = true,
            "you must deploy l2 Nft First"
        );
        require(!mintedTokens[_NFTAddr][_tokenId], "Token ID already minted");

        ERC721X deployed = L1L2AddrToMapping[_NFTAddr];
        //Interactions
        uint256 L2TokenId = deployed.safeMint(msg.sender, _uri);
        //Effects
        L1ToL2TokenId[_tokenId] = L2TokenId;
        mintedTokens[_NFTAddr][_tokenId] = true;
        return true;
    }
```

However, when burning a token to claim the original NFT, the metadata is not deleted:

```javascript
function _burnERC721(ERC721X _L2ERC721Addr, uint256 _L1tokenId) internal {
        uint256 L2TokenId = L1ToL2TokenId[_L1tokenId];
        _L2ERC721Addr.burn(L2TokenId);
    }

    function burnERC721(ERC721X _L2ERC721Addr, uint256 _L1tokenId) public {
        uint256 _type = tokenType[_L1tokenId][address(_L2ERC721Addr)];
        require(_type == 1, "wrong tokenType");

        address L1Collection = _L2ERC721Addr.L1CollectionAddress();
        require(mintedTokens[L1Collection][_L1tokenId], "Token ID not minted");

        uint256 L2TokenId = L1ToL2TokenId[_L1tokenId];
        address tokenOwner = _L2ERC721Addr.ownerOf(L2TokenId);
        require(tokenOwner == msg.sender, "not owner");

        mintedTokens[L1Collection][_L1tokenId] = false;

        _burnERC721(_L2ERC721Addr, _L1tokenId);

        encodeBurnerMetadata(_L1tokenId, L1Collection, tokenOwner);
        sendBurnerMetaDataX(msg.sender);
    }
```

Which allows anyone to get a free NFT on the L2 by:
Stake NFT -> Mint it on L2 -> Burn it, claim the original NFT -> Mint again on the L2 with the old meta data

## Impact

Malicious users are able to mint a free token on the L2.

## Recommendation

Delete the metadata or just set it to the default values.

# [L-01] Pragma set as ^0.8.14 which can lead to problems on Optimism and other L2s

## Summary

Pragma has been set to `^0.8.14` but this can lead to problems when deploying on Optimism as it currently is not compatible with 0.8.20 and newer.

## Vulnerability Detail

Contracts compiled with those versions will result in a nonfunctional or potentially damaged version that won't behave as expected.

By default, the compiler will use the latest version available, which means that contracts will be compiled with the 0.8.20 version. This can result in broken code when deployed on the Optimism network.

The reason for this is the lack of support for the `PUSH0` opcode.

You can [check the table here](https://community.optimism.io/docs/developers/build/differences/) for more information.

## Recommendation

Specify the pragma version. `0.8.19` is good.

# [L-02] Single-step process for critical ownership transfer is risky

## Summary

`AS_ERC20` and `ERC721X` are inheriting `Ownable` by OZ. However, a single-step process for critical ownership transfer is risky due to possible human error which could result in locking all the functions that use the onlyOwner modifier.

For example, an incorrect address, for which the private key is not known, could be passed accidentally.

## Recommendation

Implement the change of ownership in 2 steps:

- Approve a new address as a pendingOwner
- A transaction from the pendingOwner address claims the pending ownership change.

This mitigates the risk because if an incorrect address is used in step (1) then it can be fixed by re-approving the correct address. Only after a correct address is used in step (1) can step (2) happen and complete the ownership change.

This can be done with OZ's [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol)

# [L-03] Hardcoded values for `crossDomainMessengerAddr` and `xorigin` in L1ArtStakes.sol

## Summary

The values of variables `crossDomainMessengerAddr` and `xorigin` which are responsible for delivering the messages between the L1 and L2 network are hardcoded which will result in the contracts not working when they are deployed on another chain.

`crossDomainMessengerAddr` is currently hardcoded to `0x5086d1eEF304eb5284A0f6720f79403b4e9bE294` which is the crossDomainMessenger on Goerli and `xorigin` is hardcoded to the instance of the L2 contract on the testnet OP network.

In L2ArtStakes.sol only the `crossDomainMessengerAddr` is hardcoded to the mainnet Optimism cross-domain messenger.

## Recommendation

Create a functions `setCrossDomainMessengerAddr` and `setXOrigin` accesible only by the deployer of the L1 and L2 contracts.
