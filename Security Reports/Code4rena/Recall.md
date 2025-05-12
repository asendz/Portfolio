# Recall: Intelligence network for AI agents - Findings Report

## Table of contents
- ### High Risk Findings
    - #### H-01. Wrong use of asset source in SubnetActorManagerFacet::leave() function
    - #### H-02. Incorrect loop indexing in LibAddressStakingReleases.compact() leads to unprocessed pending releases

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://code4rena.com/audits/2025-02-recall)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2

# High Risk Findings

## <a id='H-01'></a>H-01. Wrong use of asset source in SubnetActorManagerFacet::leave() function
## Finding description and impact

In the `leave()` function, when a validator leaves a pre-bootstrapped subnet, the genesis balance (initial funds contributed to the subnet) is refunded using `s.collateralSource.transferFunds` instead of `s.supplySource.transferFunds` as used in the `preRelease()` function. Since the genesis funds are originally locked via the supplySource, refunding them from the collateralSource is inconsistent.

This misrouting results in wrong accounting, and even sending the wrong token type, if collateral and supply tokens are different, which is entirely possible, given the current code implementation.

### Proof of Concept

Let's first start with distinguishing between the `collateralSource` and `supplySource`. They are both defined in the `SubnetActorStorage` struct in `LibSubnetActorStorage.sol`:

```
/// @notice subnet supply strategy.
Asset supplySource;
/// @notice subnet collateral token strategy, only used for collateral based permission mode.
Asset collateralSource;
```

The intention is that a subnet should be able to handle different collateral and supply sources, which is evident by many of the functions in the code. Examples:  
Constructor in `SubnetActorDiamond.sol`:

```
constructor(IDiamond.FacetCut[] memory _diamondCut, ConstructorParams memory params, address owner) {
...
    s.supplySource = params.supplySource;
    s.collateralSource = params.collateralSource;
...
}
```

Function `transferFunds` in `AssetHelper.sol`:

```
function transferFunds(
    Asset memory asset,
    address payable recipient,
    uint256 value
) internal returns (bool success, bytes memory ret) {
    if (asset.kind == AssetKind.Native) {
        success = sendValue(payable(recipient), value);
        return (success, EMPTY_BYTES);
    } else if (asset.kind == AssetKind.ERC20) {
        return ierc20Transfer(asset, recipient, value);
    }
}
```

Function `fundWithToken` in `GatewayManagerFacet.sol`, test `createTokenSubnet` in `GatewayDiamondToken.t.sol`, and others.

The issue is that in `SubnetActorManagerFacet::leave()`, collateral source is used instead of supply source.

Let's first see where `supplySource` is used correctly:

```
function preFund(uint256 amount) external payable {
    if (amount == 0) {
        revert NotEnoughFunds();
    }

    if (s.bootstrapped) {
        revert SubnetAlreadyBootstrapped();
    }

    s.supplySource.lock(amount); <---

    if (s.genesisBalance[msg.sender] == 0) {
        s.genesisBalanceKeys.push(msg.sender);
    }

    s.genesisBalance[msg.sender] += amount; <---
    s.genesisCircSupply += amount; <---
}
```

```
function preRelease(uint256 amount) external nonReentrant {

    if (amount == 0) {
        revert NotEnoughFunds();
    }

    if (s.bootstrapped) {
        revert SubnetAlreadyBootstrapped();
    }

    s.supplySource.transferFunds(payable(msg.sender), amount); <---

    if (s.genesisBalance[msg.sender] < amount) {
        revert NotEnoughBalance();
    }

    s.genesisBalance[msg.sender] -= amount;
    s.genesisCircSupply -= amount;

    if (s.genesisBalance[msg.sender] == 0) {
        LibSubnetActor.rmAddressFromBalanceKey(msg.sender);
    }
}
```

As you can see, these are the function used to add and remove `genesisBalance` which is initial supply tokens added to a pre-bootstrapped subnet's initial circulating supply.

Now, let's see the issue in the `leave()` function:

```
function leave() external nonReentrant whenNotPaused notKilled {
...
// unrelated functionality
...

    if (!s.bootstrapped) {
        // check if the validator had some initial balance and return it if not bootstrapped
        uint256 genesisBalance = s.genesisBalance[msg.sender]; <---

        if (genesisBalance != 0) {

            delete s.genesisBalance[msg.sender];
            s.genesisCircSupply -= genesisBalance;
            LibSubnetActor.rmAddressFromBalanceKey(msg.sender);

            s.collateralSource.transferFunds(payable(msg.sender), genesisBalance); <--- !

        }
...
// unrelated functionality
```

As you can see, if the validator had some `genesisBalance`(which was funded with the `supplySource`), we intend to transfer it back to him:

```
s.collateralSource.transferFunds(payable(msg.sender), genesisBalance);
```

However, instead of using `supplySource`, the function uses `collateralSource` which is inconsistent and can cause accounting errors or even sending the wrong token type which can lead to fund loss if the collateral token is more expensive than the supply token.

### Recommended mitigation steps

Change to:

```
s.supplySource.transferFunds(payable(msg.sender), genesisBalance);
```

## <a id='H-02'></a>H-02. Incorrect loop indexing in LibAddressStakingReleases.compact() leads to unprocessed pending releases
## Finding description and impact
The root cause of this issue is the use of an absolute index (`self.startIdx`) in the loop condition of the `compact()` function in `LibAddressStakingReleases`, while `self.length` represents the number of pending releases.

If there are more processed releases than pending releases, the following will be true: `self.startIdx > self.length`.

This will cause the `compact()` function to never enter the processing loop:

```
function compact(AddressStakingReleases storage self) internal returns (uint256, uint16) {
    uint16 length = self.length;
    if (self.length == 0) {
        revert NoCollateralToWithdraw();
    }

    uint16 i = self.startIdx; // <-- 'i' is the absolute index, e.g., 100 after previous processing

    uint16 newLength = length;
    uint256 amount;
    while (i < length) { // <-- ! // Incorrect: compares absolute index (e.g., 100) with pending count (15)
        StakingRelease memory release = self.releases[i];

        // releases are ordered ascending by releaseAt, no need to check
        // further as they will still be locked.
        if (release.releaseAt > block.number) {
            break;
        }

        amount += release.amount;
        delete self.releases[i];

        unchecked {
            ++i;
            --newLength;
        }
    }

    self.startIdx = i; // <-- Updates startIdx to the final value of 'i'
    self.length = newLength; // Updates the length to the number of pending releases

    return (amount, newLength);
}
```

This prevents the contract from processing and releasing collateral that should be claimable, locking validator funds and breaking the core logic of the `compact` function.

Since `startIdx` only increases over time (as it is never reset), this issue becomes more likely as the absolute index grows larger and at some point it becomes inevitable.

**Note**: `compact` is called in the `claim` function that is critical for the validator collateral claiming flow.

### Proof of Concept

Let's go over the following scenario:

- 115 releases to be processed, 100 of them available to claim at the current release time.
- `compact` function is called, `i = startIdx = 0`, `newLength = length = 115`
- `compact` processes the 100 available releases, the loop breaks and this is executed:

```
self.startIdx = i; // = 100 (it was incremented 100 times)
self.length = newLength; // = 15 (it was decremented 100 times - number of pending releases)
```

Now, on the next iteration, let's say for simplicity the number of releases stays the same, and time has passed so now they are available for claiming. We have this:

- `startIdx = 100`, `self.length = 15`
- The loop starts with `i = 100` and checks `while (i < 15)`, which immediately fails

**Outcome**:

- The loop body does not execute.
- Pending releases (indices 100 to 114 in absolute terms) are never processed.

This discrepancy between the absolute index and the relative count prevents claimable collateral from being released.

This is extremely likely to happen since the absolute index of processed releases only grows - it never decreases. Therefore, it'll reach a point of its value being 100, 1000, 10_000, etc.

At some point, the value will be so big that the pending releases will inevitably be of much lower value than the processed releases, leading to locked validator collateral that should be available to claim.

### Recommended mitigation steps

Adjust the loop condition. For example, change the condition to:

```
while (i < self.startIdx + self.length)
```