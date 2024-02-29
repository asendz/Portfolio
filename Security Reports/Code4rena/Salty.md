# What is Salty.io?
Salty.IO is a Decentralized Exchange on Ethereum which uses Automatic Atomic Arbitrage (AAA) to generate yield and provide Zero Fees on all swaps.

With AAA, market inefficiencies are arbitraged at swap time to create profits - which are then distributed to liquidity providers and stakers and used to form Protocol Owned Liquidity (POL) for the DAO.

Additionally, Salty.IO provides USDS, an overcollateralized ERC20 stablecoin native to the protocol which uses WBTC/WETH LP as collateral.

The exchange is 100% decentralized at launch - with all parameters, regional exclusions, whitelisting, and contracts controlled by the DAO itself.

[Link to the contest](https://code4rena.com/audits/2024-01-saltyio)

# Issues found by me

| Severity | Title                                                                                        | 
| :------- | :------------------------------------------------------------------------------------------- | 
| High     | User can become un-liquidatable by perpetually extending his cooldown |
| Medium     | Attackers can brick proposals by creating fake confirmation proposals | 
| Medium     | SALT emissions can't be fully emitted if performUpkeep() is called too often due to a precision loss | 
| Medium     | When a pool gets unwhitelisted, all of the unclaimed accrued rewards will be lost | 

# 1. User can become un-liquidatable by perpetually extending his cooldown

## Impact
A malicious user can avoid liquidation by perpetually extending his cooldown period which will result in bad debt for the protocol.
## Proof of Concept
Let's take a look at the `liquidateUser` function in CollateralAndLiquidity.sol:
```solidity
function liquidateUser( address wallet ) external nonReentrant
                {
                require( wallet != msg.sender, "Cannot liquidate self" );

                // First, make sure that the user's collateral ratio is below the required level
                require( canUserBeLiquidated(wallet), "User cannot be liquidated" );

                uint256 userCollateralAmount = userShareForPool( wallet, collateralPoolID );

                // Withdraw the liquidated collateral from the liquidity pool.
                // The liquidity is owned by this contract so when it is withdrawn it will be reclaimed by this contract.
                (uint256 reclaimedWBTC, uint256 reclaimedWETH) = pools.removeLiquidity(wbtc, weth, userCollateralAmount, 0, 0, totalShares[collateralPoolID] );

                // Decrease the user's share of collateral as it has been liquidated and they no longer have it.
                _decreaseUserShare( wallet, collateralPoolID, userCollateralAmount, true );

//unrelated functionality below
                }
```
We're calling the `_decreaseUserShare` function in the StakingRewards.sol contract with `true` passed as `useCooldown`:
```solidity
        function _decreaseUserShare( address wallet, bytes32 poolID, uint256 decreaseShareAmount, bool useCooldown ) internal
                {
                require( decreaseShareAmount != 0, "Cannot decrease zero share" );

                UserShareInfo storage user = _userShareInfo[wallet][poolID];
                require( decreaseShareAmount <= user.userShare, "Cannot decrease more than existing user share" );

                if ( useCooldown )
                if ( msg.sender != address(exchangeConfig.dao()) ) // DAO doesn't use the cooldown
                        {
                        require( block.timestamp >= user.cooldownExpiration, "Must wait for the cooldown to expire" );

                        // Update the cooldown expiration for future transactions
                        user.cooldownExpiration = block.timestamp + stakingConfig.modificationCooldown();
                        }
//unrelated functionality below
}
```
As you can see, we'll enter the 1st if, and given that the `msg.sender` is a random liquidator, we'll enter the 2nd if as well. Which gets us to this:
```solidity
        require( block.timestamp >= user.cooldownExpiration, "Must wait for the cooldown to expire" );
```
This means the user can't get liquidated if the cooldown hasn't expired. The default value of the cooldown is 1 hour:
```solidity
        // Minimum time between increasing and decreasing user share in SharedRewards contracts.
        // Prevents reward hunting where users could frontrun reward distributions and then immediately withdraw.
        // Range: 15 minutes to 6 hours with an adjustment of 15 minutes
        uint256 public modificationCooldown = 1 hours;
```

Now, considering that the cooldown gets extended both in `_decreaseUserShare` and `_increaseUserShare`:
```solidity
        function _increaseUserShare(/*params*/) internal
                {
                require( poolsConfig.isWhitelisted( poolID ), "Invalid pool" );
                require( increaseShareAmount != 0, "Cannot increase zero share" );
                UserShareInfo storage user = _userShareInfo[wallet][poolID];
                if ( useCooldown ) {
                if ( msg.sender != address(exchangeConfig.dao()) ) // DAO doesn't use the cooldown
                        {
                        require( block.timestamp >= user.cooldownExpiration, "Must wait for the cooldown to expire" );

                        // Update the cooldown expiration for future transactions
                        user.cooldownExpiration = block.timestamp + stakingConfig.modificationCooldown();
                        }
                }
//unrelated functionality below
}
```
A malicious user could:
- Set up a keeper type of bot that increases/decreases his share with small amounts immediately after or right when the cooldown expires. Keeping himself in a perpetual cooldown state making himself unable to be liquidated.
- Or monitoring the mem pool and frontrunning a liquidation transaction with an increase/decrease of his shares, making himself unable to be liquidated.
## Tools Used
Manual review
## Recommended Mitigation Steps
Skip the cooldown check when calling from `liquidateUser` by passing `false` for the `useCooldown` parameter:
```solidity
_decreaseUserShare( wallet, collateralPoolID, userCollateralAmount, false);
```

# 2. Attackers can brick proposals by creating fake confirmation proposals

## Impact
Malicious users can brick proposals like `SET_CONTRACT` and `SET_WEBSITE_URL` by creating another proposal that end with "_confirm".
## Proof of Concept
When finalizing approval proposals, the following function from the DAO.sol contract is called:
```solidity
function _executeApproval( Ballot memory ballot ) internal
                {
//unrelated functionality
// Once an initial setContract proposal passes, it automatically starts a second confirmation ballot (to prevent last minute approvals)
                else if ( ballot.ballotType == BallotType.SET_CONTRACT )
                        proposals.createConfirmationProposal( string.concat(ballot.ballotName, "_confirm"), BallotType.CONFIRM_SET_CONTRACT, ballot.address1, "", ballot.description );

                // Once an initial setWebsiteURL proposal passes, it automatically starts a second confirmation ballot (to prevent last minute approvals)
                else if ( ballot.ballotType == BallotType.SET_WEBSITE_URL )
                        proposals.createConfirmationProposal( string.concat(ballot.ballotName, "_confirm"), BallotType.CONFIRM_SET_WEBSITE_URL, address(0), ballot.string1, ballot.description );
}
```
which in turn tries to create a confirmation proposal via this function:
```solidity
function createConfirmationProposal(
                string calldata ballotName,
                BallotType ballotType,
                address address1,
                string calldata string1,
                string calldata description
                ) external returns (uint256 ballotID)
                {
                require( msg.sender == address(exchangeConfig.dao()), "Only the DAO can create a confirmation proposal" );

                return _possiblyCreateProposal( ballotName, ballotType, address1, 0, string1, description );
                }

```
which calls `_possiblyCreateProposal` in Proposals.sol
```solidity
function _possiblyCreateProposal(
                string memory ballotName,
                BallotType ballotType,
                address address1,
                uint256 number1,
                string memory string1,
                string memory string2
                ) internal returns (uint256 ballotID)
                {
//unrelated functionality
                require( openBallotsByName[ballotName] == 0, "Cannot create a proposal similar to a ballot that is still open" );

                require( openBallotsByName[ string.concat(ballotName, "_confirm")] == 0, "Cannot create a proposal for a ballot with a secondary confirmation" );
```
However, nothing stops a regular user from creating a proposal that ends with "_confirm" as soon as he sees a proposal for `SET_CONTRACT` or `SET_WEBSITE_URL`.

For example, if the proposal for setting a contract or a website is called "ProposalName"

He'd create a brand new proposal that ends with "_confirm" i.e "ProposalName_confirm" and these checks will pass:
```solidity
require( openBallotsByName[ballotName] == 0, "Cannot create a proposal similar to a ballot that is still open" );

require( openBallotsByName[ string.concat(ballotName, "_confirm")] == 0, "Cannot create a proposal for a ballot with a secondary confirmation" );
```
The 1st one will pass because there won't be a proposal with the same name. The 2nd one will also pass because there is no proposal named "ProposalName_confirm_confirm"(after concatenating another _confirm at the end).

However, after that, the real confirmation proposal will fail on this check:
```solidity
require( openBallotsByName[ string.concat(ballotName, "_confirm")] == 0, "Cannot create a proposal for a ballot with a secondary confirmation" );
```
Because a proposal with the name "ProposalName_confirm" already exists so the value won't be 0.
## Tools Used
Manual review
## Recommended Mitigation Steps
You could prevent users from creating proposals that end with "_confirm".
You can use a helper function like that that returns true if the proposal doesn't end with "_confirm":
```solidity
function isNotConfirmSuffix(string memory str) internal pure returns (bool) {
  bytes memory strBytes = bytes(str);
  if (strBytes.length < 8) {
      return true; // String is too short to have "_confirm" as a suffix
  }

  bytes memory confirmSuffix = bytes("_confirm");
  for(uint i = 0; i < 8; i++) {
      if (strBytes[strBytes.length - 8 + i] != confirmSuffix[i]) {
          return true; // Suffix does not match "_confirm"
      }
  }

  return false; // Suffix matches "_confirm"
}
```
Then, when creating a proposal:
```solidity
require(isNotConfirmSuffix(proposalName), "Name ends with _confirm");
```

# 3. SALT emissions can't be fully emitted if performUpkeep() is called too often due to a precision loss

## Impact
SALT emissions can't be fully emitted, due to a precision loss, if performUpkeep() is called too often, leading to stuck SALT in the Emissions.sol contract.
## Proof of Concept
Let's take a look at the `performUpkeep` function in the Emission contract:
```solidity
        function performUpkeep(uint256 timeSinceLastUpkeep) external
                {
                require( msg.sender == address(exchangeConfig.upkeep()), "Emissions.performUpkeep is only callable from the Upkeep contract" );

                if ( timeSinceLastUpkeep == 0 )
                        return;

                // Cap the timeSinceLastUpkeep at one week (if for some reason it has been longer).
                // This will cap the emitted rewards at a default of 0.50% in this transaction.
                if ( timeSinceLastUpkeep >= MAX_TIME_SINCE_LAST_UPKEEP )
                        timeSinceLastUpkeep = MAX_TIME_SINCE_LAST_UPKEEP;

                uint256 saltBalance = salt.balanceOf( address( this ) );

                // Target a certain percentage of rewards per week and base what we need to distribute now on how long it has been since the last distribution
                uint256 saltToSend = ( saltBalance * timeSinceLastUpkeep * rewardsConfig.emissionsWeeklyPercentTimes1000() ) / ( 100 * 1000 weeks );
                if ( saltToSend == 0 )
                        return;

                // Send the emissions to saltRewards
                salt.safeTransfer(address(saltRewards), saltToSend);
                }
```
and more specifically the calculation of `saltToSend`:
```solidity
uint256 saltBalance = salt.balanceOf(address(this));

uint256 saltToSend = (saltBalance * timeSinceLastUpkeep * rewardsConfig.emissionsWeeklyPercentTimes1000()) / (100 * 1000 weeks);
```
Let's consider this scenario:
- `timeSinceLastUpkeep`: 12 seconds - this has been the consistent avg block time since Ethereum went to Proof of Stake
- emissionsWeeklyPercentTimes1000 = 500 (default 0.5% rate times 1000)
- Denominator: 100 * 1000 * 604800 (number of seconds in a week, multiplied by 1000 and 100)
- saltBalance = 10_000_000

Then we'd have:
saltToSend = (10_000_000 * 12 * 500) / 60_480_000_000 =
= 60_000_000_000 / 60_480_000_000 = ~0.99 which rounds down to 0.

This leads to 10_000_000 units of SALT that will be stuck in the contract.

The effects of this will be extrapolated if the protocol launches on a L2 like Arbitrum, for example, where average block time is 0.3s(over the last 5000 blocks, this can be checked on [Arbiscan](https://arbiscan.io/)).

On such an L2, the stuck saltBalance could be 40 times bigger(12sec/0.3s) = 400_000_000 SALT.

This issue will present itself either because of a malicious actor trying to consciously stuck the salt balance, or a genuine user who wants to farms the incentives for calling `performUpkeep`.
## Tools Used
Manual review
## Recommended Mitigation Steps
A simple fix would be if `saltToSend == 0` then revert instead of return:
```solidity
                if ( saltToSend == 0 )
                        revert;
```
This will lead to the transaction reverting, which will lead to `lastUpkeepTimeEmissions` to ***not*** update in the `step6()` function. By keeping the `lastUpkeepTimeEmissions` at the value which resulted last in sending actual balance of salt, you're guaranteeing that there won't be such precision loss caused by too frequent calls to `performUpkeep()`.

Reverting the transaction should not cause any problems to the rest of the `performUpkeep()` function of the Upkeep.sol contract because the calls are wrapped in try/catch and there will be just an event emitted.

# 4. When a pool gets unwhitelisted, all of the unclaimed accrued rewards will be lost

## Impact
When a pool gets unwhitelisted, all of the accrued since the last call to `performUpkeep()` rewards will be lost.
## Proof of Concept
Let's see the code of the `unwhitelistPool` function in PoolsConfig.sol:
```solidity
function unwhitelistPool( IPools pools, IERC20 tokenA, IERC20 tokenB ) external onlyOwner
                {
                bytes32 poolID = PoolUtils._poolID(tokenA,tokenB);

                _whitelist.remove(poolID);
                delete underlyingPoolTokens[poolID];

                // Make sure that the cached arbitrage indicies in PoolStats are updated
                pools.updateArbitrageIndicies();

                emit PoolUnwhitelisted(address(tokenA), address(tokenB));
                }
```
As you can see, the pool is only removed from the whitelist and then its tokens are deleted from the mapping.

However, the rewards are processed as pending only when the `performUpkeep()` function in the Upkeep.sol contract is called. Let's take a look more specifically at `step8()` in the Upkeep contract:
```solidity
        function step8() public onlySameContract
                {
                uint256 timeSinceLastUpkeep = block.timestamp - lastUpkeepTimeRewardsEmitters;

                saltRewards.stakingRewardsEmitter().performUpkeep(timeSinceLastUpkeep);
                saltRewards.liquidityRewardsEmitter().performUpkeep(timeSinceLastUpkeep);

                lastUpkeepTimeRewardsEmitters = block.timestamp;
                }
```
Here, we call `performUpkeep()` in the RewardsEmitter contract:
```solidity
        function performUpkeep( uint256 timeSinceLastUpkeep ) external
                {
                require( msg.sender == address(exchangeConfig.upkeep()), "RewardsEmitter.performUpkeep is only callable from the Upkeep contract" );

                if ( timeSinceLastUpkeep == 0 )
                        return;

                bytes32[] memory poolIDs;

                 if ( isForCollateralAndLiquidity )
                        {
                        // For the liquidityRewardsEmitter, all pools can receive rewards
                        poolIDs = poolsConfig.whitelistedPools(); <---
                        }
                 else
                        {
                        // The stakingRewardsEmitter only distributes rewards to those that have staked SALT
                        poolIDs = new bytes32[](1);
                        poolIDs[0] = PoolUtils.STAKED_SALT;
                        }

                // Cap the timeSinceLastUpkeep at one day (if for some reason it has been longer).
                // This will cap the emitted rewards at a default of 1% in this transaction.
                if ( timeSinceLastUpkeep >= MAX_TIME_SINCE_LAST_UPKEEP )
        timeSinceLastUpkeep = MAX_TIME_SINCE_LAST_UPKEEP;

                // These are the AddedRewards that will be sent to the specified StakingRewards contract
                AddedReward[] memory addedRewards = new AddedReward[]( poolIDs.length );

                // Rewards to emit = pendingRewards * timeSinceLastUpkeep * rewardsEmitterDailyPercent / oneDay
                uint256 numeratorMult = timeSinceLastUpkeep * rewardsConfig.rewardsEmitterDailyPercentTimes1000();
                uint256 denominatorMult = 1 days * 100000; // simplification of numberSecondsInOneDay * (100 percent) * 1000

                uint256 sum = 0;
                for( uint256 i = 0; i < poolIDs.length; i++ )
                        {
                        bytes32 poolID = poolIDs[i];

                        // Each pool will send a percentage of the pending rewards based on the time elapsed since the last send
                        uint256 amountToAddForPool = ( pendingRewards[poolID] * numeratorMult ) / denominatorMult;

                        // Reduce the pending rewards so they are not sent again
                        if ( amountToAddForPool != 0 )
                                {
                                pendingRewards[poolID] -= amountToAddForPool;

                                sum += amountToAddForPool;
                                }

                        // Specify the rewards that will be added for the specific pool
                        addedRewards[i] = AddedReward( poolID, amountToAddForPool );
                        }

                // Add the rewards so that they can later be claimed by the users proportional to their share of the StakingRewards derived contract.
                stakingRewards.addSALTRewards( addedRewards );
                }
```
As you can see, the rewards for providing liquidity are only added for whitelisted pools:
```solidity
if ( isForCollateralAndLiquidity )
                        {
                        // For the liquidityRewardsEmitter, all pools can receive rewards
                        poolIDs = poolsConfig.whitelistedPools(); <---
                        }
```
This makes sense, however, if we unwhitelist a pool, and `performUpkeep` was last called 1 hour ago or 12 hours ago, there will be accrued rewards that won't be accounted for and will be completely lost.
## Tools Used
Manual review
## Recommended Mitigation Steps
Call the upkeep function before removing the pool from the whitelist to ensure all rewards have been accounted for and paid:
```solidity
function unwhitelistPool( IPools pools, IERC20 tokenA, IERC20 tokenB ) external onlyOwner
                {
                bytes32 poolID = PoolUtils._poolID(tokenA,tokenB);

               address(Upkeep).performUpkeep(); <---

                _whitelist.remove(poolID);
                delete underlyingPoolTokens[poolID];

                // Make sure that the cached arbitrage indicies in PoolStats are updated
                pools.updateArbitrageIndicies();

                emit PoolUnwhitelisted(address(tokenA), address(tokenB));
                }
```