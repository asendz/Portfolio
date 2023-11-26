# What is SteadeFi?

Steadefi provides vaults with automated risk management for earning leveraged yields effectively and passively in bull, crab and bear markets. With lending and leveraged delta long and neutral stategies, Steadefi's vaults cater to different risk/reward strategies to the best yield-generating DeFi protocols.

[Link to the contest](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# Issues found by me

| Severity | Title                                                                                                       | Link                                                         |
| :------- | :---------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- |
| High     | A malicious user can DOS the system leaving it in a perpetual status of "Deposit | [Link](https://www.codehawks.com/finding/clpcx1vtd000h5ehz7qxk938d)  |
| Medium     | Failed Compound causes DOS to the entire system                              | [Link](https://www.codehawks.com/finding/clpcx1whd001f5ehzjob4e0p5) |
| Medium     | GMXDeposit - deposit() function passes a wrong flag to gmxOracle.getLpTokenValue()         | [Link](https://www.codehawks.com/finding/clpcx1vyz000n5ehz0n577mkd)  |

# 1. A malicious user can DOS the system leaving it in a perpetual status of "Deposit"

## Summary

A malicious user can basically DOS the whole system on demand and leave it perpetually in a "Deposit" status.

For that to happen, let's look at the user flow of depositing when `addLiquidity` to GMX fails and later we'll explain how we can force this. 

The user flow would be:
1. User calls `deposit()` on GMXVault, which sets `Vault.Status = Deposit`
2. Borrowing of assets and transfer of tokens happens, then `addLiquidity` is called to GMX
3. If `addLiquidity` fails then `afterDepositCancellation` on GMXCallback is called, which calls `processDepositCancellation` in GMXDeposit

This can be easily traced in the [deposit sequence](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/docs/sequences/strategy-gmx-deposit-sequence-detailed.md).

## Vulnerability Detail

Let's take a look at the ``processDepositCancellation`` in GMXDeposit:
```solidity
function processDepositCancellation(
    GMXTypes.Store storage self
  ) external {
    GMXChecks.beforeProcessDepositCancellationChecks(self);

    // Repay borrowed assets
    GMXManager.repay(
      self,
      self.depositCache.borrowParams.borrowTokenAAmt,
      self.depositCache.borrowParams.borrowTokenBAmt
    );

    // Return user's deposited asset
    // If native token is being withdrawn, we convert wrapped to native
    if (self.depositCache.depositParams.token == address(self.WNT)) {
      self.WNT.withdraw(self.WNT.balanceOf(address(this)));

      (bool success, ) = self.depositCache.user.call{value: address(this).balance}(""); <--
      require(success, "Transfer failed."); <--
    } else {
      // Transfer requested withdraw asset to user
      IERC20(self.depositCache.depositParams.token).safeTransfer(
        self.depositCache.user,
        self.depositCache.depositParams.amt
      );
    }

    self.status = GMXTypes.Status.Open;

    emit DepositCancelled(self.depositCache.user);
  }
```
As you can see if the deposit token is native, then we're going to refund the user's native token and check for success via these lines:
```solidity
      (bool success, ) = self.depositCache.user.call{value: address(this).balance}("");
      require(success, "Transfer failed.");
```
However, what a malicious user could do is deposit a native token through a smart contract that doesn't have a `receive()` function, for example, making the call always revert. In that case, the `processDepositCancellation` function can't be executed, and the status of the Vault can't be reset to Open, DOSing the system. 

The user can maximize the chance of `addLiquidity` failing by specifying a very low slippage amount in his deposit params which would be `slippage = minSlippage(of the vault)`. `minSlippage` here shouldn't be too high because otherwise, it'll be too restrictive for the users. 

Given that slippage depends on liquidity, and liquidity depends on market conditions or it can also be influenced by flash loans, for example, I believe that the chance of a malicious user succeeding in his attempts to make `addLiquidity` fail is very high.

Also, he can try numerous times with the `MINIMUM_VALUE` deposit which is currently set as 9e16 in GMXChecks. 9e16 = 0.09ETH or AVAX, depending on which blockchain. 0.09 * 12 (price of AVAX) = ~1$

## Impact

DOS of most of the functionality of the protocol. The status is set as "Deposit" when `deposit()` is called, and being unable to `processDepositCancellation` leaves the system in this status permanently. 

Functions that are impacted:

Deposit will revert:
```solidity
  function beforeDepositChecks(
    GMXTypes.Store storage self,
    uint256 depositValue
  ) external view {
    if (self.status != GMXTypes.Status.Open)
      revert Errors.NotAllowedInCurrentVaultStatus();
```

Withdraw will revert:
```solidity
  function beforeWithdrawChecks(
    GMXTypes.Store storage self
  ) external view {
    if (self.status != GMXTypes.Status.Open)
      revert Errors.NotAllowedInCurrentVaultStatus();
```

Rebalance will revert:
```solidity
  function beforeRebalanceChecks(
    GMXTypes.Store storage self,
    GMXTypes.RebalanceType rebalanceType
  ) external view {
    if (
      self.status != GMXTypes.Status.Open &&
      self.status != GMXTypes.Status.Rebalance_Open
    ) revert Errors.NotAllowedInCurrentVaultStatus();
```

Compound will revert:
```solidity
  function beforeCompoundChecks(
    GMXTypes.Store storage self
  ) external view {
    if (
      self.status != GMXTypes.Status.Open &&
      self.status != GMXTypes.Status.Compound_Failed
    ) revert Errors.NotAllowedInCurrentVaultStatus();
```

## Code Snippet

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L209

## Recommendation

You can wrap the call in a try/catch like at other places in the code. Or, you could just credit his balance in an internal mapping and let him claim his assets from another function `claimFailedDeposit`. 

# 2. Failed Compound causes DOS to the entire system   

## Summary

A failed Compound can cause a DOS to pretty much the entire system. The reason for this is because of the status set in `processCompoundCancellation` which is the last function called in the user flow where a compound action initiated by the Keeper fails for some reason. 

There could be many reasons for this - for example, a lack of liquidity which incurs much bigger slippage than what we want as a system, resulting in failed token swap and failed compound action. The system is expected to handle such cases without bricking other functionality. 

## Vulnerability Detail

Let's see the code of `processCompoundCancellation` in GMXCompound.sol:
```solidity
  function processCompoundCancellation(GMXTypes.Store storage self) external {
    GMXChecks.beforeProcessCompoundCancellationChecks(self);

    self.status = GMXTypes.Status.Compound_Failed; <--

    emit CompoundCancelled();
  }
```

As you can see, the status is set as `Compound_Failed`. I won't get into detail on how we end up using this function as the report will get very long but this is basically the last function that gets called if the keeper tries to "compound" and the compounding fails. The whole flow can be easily checked in the [detailed sequence diagrams](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/docs/sequences/strategy-gmx-compound-sequence-detailed.md) made by the SteadeFi team.

Setting the status as `Compound_Failed` bricks the functionality of very important functions like `deposit`, `withdraw`, and `rebalance`. 

Check before a deposit:
```solidity
function beforeDepositChecks(
    GMXTypes.Store storage self,
    uint256 depositValue
  ) external view {
    if (self.status != GMXTypes.Status.Open)
      revert Errors.NotAllowedInCurrentVaultStatus();

//other unrelated checks below
}
```

Check before a withdraw:
```solidity
  function beforeWithdrawChecks(
    GMXTypes.Store storage self
  ) external view {
    if (self.status != GMXTypes.Status.Open)
      revert Errors.NotAllowedInCurrentVaultStatus();

//other unrelated checks below
```

Check before a rebalance:
```solidity
  function beforeRebalanceChecks(
    GMXTypes.Store storage self,
    GMXTypes.RebalanceType rebalanceType
  ) external view {
    if (
      self.status != GMXTypes.Status.Open &&
      //this only gets set later in the process of rebalance so the functionality here will be bricked as well
      self.status != GMXTypes.Status.Rebalance_Open 
    ) revert Errors.NotAllowedInCurrentVaultStatus();
//other unrelated checks below
}
```

As you can see, these primary functions will be completely bricked because the status is not Open. The sponsor confirmed that the keepers will likely try to compound once a day, meaning this DOS can last days at a time making the system basically useless.

The only way to fix the DOS is if we manage to successfully process a compound which sets the status again to open: 
```solidity
  function processCompound(
    GMXTypes.Store storage self
  ) external {
    GMXChecks.beforeProcessCompoundChecks(self);

    self.status = GMXTypes.Status.Open;

    emit CompoundCompleted();
  }
```
However, this might not be possible immediately and the keepers likely won't try to compound again leaving the system DOSed for an undefined amount of time.

Additionally, according to the sequence diagram, it looks like we expect the status to be Open after `processCompoundCancellation`:
[Screenshot](https://prnt.sc/ZMZ5SUKxQTsJ)

## Impact

DOS of the most important functions in the protocol which could continue for days.

That is not intended behavior and will lead to frustrated users and eventually to losses for the protocol. 

## Code Snippet
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCompound.sol#L132/gmx/GMXCompound.sol#L132

## Recommendation
Set the status to open even if the compound failed: 
`self.status = GMXTypes.Status.Open;`

# 3. GMXDeposit - deposit() function passes a wrong flag to gmxOracle.getLpTokenValue()

## Summary
The deposit() function calls gmxOracle.getLpTokenValue with wrong flag for `isDeposit`. 
## Vulnerability Details
Let's take a look at this part of the code of the deposit() function:
```solidity
      _dc.depositValue = self.gmxOracle.getLpTokenValue(
        address(self.lpToken),
        address(self.tokenA),
        address(self.tokenA),
        address(self.tokenB),
        false, <--
        false
      )
```
and take a look at the `getLpTokenValue()` in GMXOracle: 
```solidity
function getLpTokenValue(
    address marketToken,
    address indexToken,
    address longToken,
    address shortToken,
    bool isDeposit, <--
    bool maximize
  ) public view returns (uint256) {
    bytes32 _pnlFactorType;

    if (isDeposit) {
      _pnlFactorType = keccak256(abi.encode("MAX_PNL_FACTOR_FOR_DEPOSITS"));
    } else {
      _pnlFactorType = keccak256(abi.encode("MAX_PNL_FACTOR_FOR_WITHDRAWALS")); <--
    }

    (int256 _marketTokenPrice,) = getMarketTokenInfo(
      marketToken,
      indexToken,
      longToken,
      shortToken,
      _pnlFactorType,
      maximize
    );

    // If LP token value is negative, return 0
    if (_marketTokenPrice < 0) {
      return 0;
    } else {
      // Price returned in 1e30, we normalize it to 1e18
      return uint256(_marketTokenPrice) / 1e12;
    }
  }
```
As you can see, we're passing `isDeposit == false` so the `_pnlFactorType` will be for withdrawals.
## Impact
I doubt this has much impact, depends on the way the GMX system utilizes it, but may cause some problems on the front-end for example, displaying a wrong pnlFactor. I still think this deserves a Low severity

## Code Snippet

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L97

## Recommendation

Pass it as true because we're depositing.