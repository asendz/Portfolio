# AI Arena - Findings Report

## Table of contents

* ### High Risk Findings

  * #### H-01. Non-transferable GameItems can be transferred with GameItems::safeBatchTransferFrom(...)
  * #### H-02. Players can mint fighters with rare attributes and restricted types via redeemMintPass
  * #### H-03. Bypassing maxRerollsAllowed and rerolling with a different fighterType
  * #### H-04. Locked fighters can be transferred and exploited, causing game server failures

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://code4rena.com/audits/2024-02-ai-arena)

# <a id='results-summary'></a>Results Summary

### Number of findings:

* High: 4
* Medium: 0
* Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. Non-transferable GameItems can be transferred with GameItems::safeBatchTransferFrom(...)

### Summary

GameItems marked as non-transferable can still be transferred due to missing checks in the `safeBatchTransferFrom` implementation.

### Vulnerability Details

While the contract correctly blocks transfers via `safeTransferFrom` for non-transferable items, it fails to override and implement necessary restrictions in `safeBatchTransferFrom`. As a result, users can use batch transfers to bypass transfer restrictions.

### Impact

Users can freely transfer GameItems that were intended to be non-transferable, violating the business logic and game design assumptions.

### Proof of Concept

See the following test:

```solidity
function testNonTransferableItemCanBeTransferredWithBatchTransfer() public {
    _fundUserWith4kNeuronByTreasury(_ownerAddress);
    _gameItemsContract.mint(0, 1);
    _gameItemsContract.adjustTransferability(0, false);
    vm.expectRevert();
    _gameItemsContract.safeTransferFrom(_ownerAddress, _DELEGATED_ADDRESS, 0, 1, "");
    
    // Bypasses the check
    uint256[] memory ids = new uint256[](1);
    ids ;
    amounts[0] = 1;
    _gameItemsContract.safeBatchTransferFrom(_ownerAddress, _DELEGATED_ADDRESS, ids, amounts, "");
}
```

### Tools Used

Manual review

### Recommendations

Override `safeBatchTransferFrom` and enforce the `transferable` check for each item.

---

## <a id='H-02'></a>H-02. Players can mint fighters with rare attributes and restricted types via redeemMintPass

### Summary

Players are allowed to provide arbitrary fighter types and DNA when redeeming mint passes, enabling minting of rare and restricted characters.

### Vulnerability Details

The contract does not bind a mint pass to specific attributes. A player can:

* Specify fighterType=1 (e.g., Dendroid) regardless of restriction
* Reuse previously known DNAs that yield rare attributes

### Impact

* Rare fighters can be minted easily
* Pseudo-random attribute generation can be manipulated

### Proof of Concept

```solidity
_fighterFarmContract.redeemMintPass(..., fighterTypes: [1], mintPassDNAs: [knownRareDna], ...);
```

### Tools Used

Manual review

### Recommendations

Tie mint pass to expected attributes using a signature or allowlist mechanism. Prevent arbitrary inputs.

---

## <a id='H-03'></a>H-03. Bypassing maxRerollsAllowed and rerolling with a different fighterType

### Summary

The `reRoll` function does not check if the provided fighterType matches the NFT's actual fighterType.

### Vulnerability Details

A user can:

* Supply a fighterType with higher `maxRerollsAllowed`
* Bypass the limit imposed on their own NFT's type

### Impact

Fighters can gain additional rerolls and change attributes using more favorable reroll limits.

### Proof of Concept

```solidity
_fighterFarmContract.reRoll(tokenId, fighterType: Dendroid);
```

### Tools Used

Manual review

### Recommendations

Validate that the input `fighterType` matches the actual fighter's stored type.

---

## <a id='H-04'></a>H-04. Locked fighters can be transferred and exploited, causing game server failures

### Summary

Transfer restrictions are bypassed via inherited ERC721 functions (e.g., `safeTransferFrom(..., data)`), enabling movement of locked fighters.

### Vulnerability Details

This transfer invalidates assumptions in the game logic:

* Server cannot reduce points from transferred fighters (unstoppable)
* Transferred fighters may affect other players' ability to exit losing zones

### Impact

Game server logic is compromised. Stuck transactions. Fair play undermined.

### Proof of Concept

```solidity
// Use safeTransferFrom with data argument
_fighterFarmContract.safeTransferFrom(from, to, tokenId, "");
```

### Tools Used

Manual review

### Recommendations

Enforce transferability in `_beforeTokenTransfer`, covering all transfer paths inherited from ERC721.
