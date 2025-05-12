# MagicSea: DEX - Findings Report

## Table of contents
- ### High Risk Findings
    - #### H-01. Users can't vote because of a wrong check in BribeRewarder::_modify()
    - #### H-02. BribeRewarder.sol allows reward manipulation by malicious users
- ### Medium Risk Findings
    - #### M-01. Wrong check in _requireOnlyOperatorOrOwnerOf in MlumStaking.sol leading to anyone being able to add to someone else's position
    - #### M-02. Voter.sol::onRegister() - anyone can register a fake rewarders and DOS the registration of new legit ones



# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://audits.sherlock.xyz/contests/437)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Users can't vote because of a wrong check in BribeRewarder::_modify()

## Summary
The current implementation mistakenly checks if the Voter contract is an owner of the user's staking position leading to a revert and inability of the users to vote.

## Vulnerability Details

Let's check the user flow to cast a vote.
A user calls `Voter.sol::vote()`:
```solidity
function vote(uint256 tokenId, address[] calldata pools, uint256[] calldata deltaAmounts) external {
//unrelated functionality
 
       // check ownership of tokenId
        if (_mlumStaking.ownerOf(tokenId) != msg.sender) {
            revert IVoter__NotOwner();
        }

//unrelated functionality

             //notify the rewarders about the casted votes
            _notifyBribes(_currentVotingPeriodId, pool, tokenId, deltaAmount);
}
```
As you can see, the function checks if the user(msg.sender) is the owner of the tokenId of the staking position and later the _notifyBribes function is called:
```solidity
    function _notifyBribes(uint256 periodId, address pool, uint256 tokenId, uint256 deltaAmount) private {

        IBribeRewarder[] storage rewarders = _bribesPerPriod[periodId][pool];

        for (uint256 i = 0; i < rewarders.length; ++i) {
            if (address(rewarders[i]) != address(0)) {
                rewarders[i].deposit(periodId, tokenId, deltaAmount);
                _userBribesPerPeriod[periodId][tokenId].push(rewarders[i]);
            }
        }
    }
```
And the function calls the deposit() function for each rewarder so that it notifies them of the casted votes and so the user is able to collect his rewards. BribeRewarder.sol::deposit():
```solidity
    function deposit(uint256 periodId, uint256 tokenId, uint256 deltaAmount) public onlyVoter {
        _modify(periodId, tokenId, deltaAmount.toInt256(), false);

        emit Deposited(periodId, tokenId, _pool(), deltaAmount);
    }
```
At this point the msg.sender is the Voter.sol contract and this is additionally verified by the modifier onlyVoter that enables only the Voter contract to call this function. Then _modify() is called:
```solidity
    function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
        private
        returns (uint256 rewardAmount)
    {

        if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
            revert BribeRewarder__NotOwner();
        }
//unrelated functionality
}
```
However, as you can see, the function checks if the msg.sender is the owner of the tokenId(staking position), and if it's not, the transaction reverts.

But the msg.sender here is the Voter contract, and the owner is the user, so the transaction will revert even though the user should be able to vote.
## Impact
Users are unable to vote.
## Tools Used

Manual review

## Recommendations
The check is unnecessary in the vote -> deposit flow as we're verifying that the user is the owner of tokenId in the vote() function as shown.

The check is only needed in the claim() function so it should be moved there and deleted from _modify like that:
```solidity
    function claim(uint256 tokenId) external override {
        //get the latest finished period
        uint256 endPeriod = IVoter(_caller).getLatestFinishedPeriod();
        uint256 totalAmount;

        if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) { <----
            revert BribeRewarder__NotOwner();
        }

        // calc emission per period cause every period can every other durations
        for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {

            totalAmount += _modify(i, tokenId, 0, true);
        }

        emit Claimed(tokenId, _pool(), totalAmount);
    }
```
## <a id='H-02'></a>H-02. BribeRewarder.sol allows reward manipulation by malicious users

## Summary
A vulnerability in the BribeRewarder contract allows a malicious user to manipulate reward calculations, preventing other users from receiving the correct amount of rewards.

## Vulnerability Details

A malicious user can exploit the claim function to repeatedly update the lastUpdateTimestamp.

This manipulation affects the _calculateRewards function, which uses the lastUpdateTimestamp to determine the rewards.

When another user tries to vote and deposit, the rewards calculation will be based on a much shorter duration than expected, significantly reducing the rewards they receive.

Consider the following scenario:

- There's a voting period1 incentivized for bribing where a malicious user1 has participated. Let's say that this period lasted 10 days.
- Next period2 starts, user1 claims his rewards and with doing that he sets the lastUpdateTimestamp to the beginning of period2. Let's say that period2 lasts from 10th to the 20th day so again 10 days
- On the day15 i.e day 5 of period2, a legitimate user2 deposits


Because of the way the rewards are calculated in _calculateRewards he should receive rewards for 5 days:
```solidity
        uint256 timestamp = block.timestamp > endTime ? endTime : block.timestamp;

        return timestamp > lastUpdateTimestamp ? (timestamp - lastUpdateTimestamp) * emissionsPerSecond : 0;
```
where block.timestamp is not > than endTime since period2 is still going. so timestamp = block.timestamp

in the 2nd condition:
timestamp is > lastUpdatedTimestamp then the rewards he should get is (block.timestamp - lastUpdateTimestamp(currently beginning of the period2) * emissions). So he should receive rewards for 5 days.

However, the malicious user1 can continuously claim rewards even if he doesn't have anything to claim, which will update lastUpdateTimestamp so he can always update it to the most recent timestamp.

Therefore, when user2 claims, his reward will be calculated as block.timestamp - lastUpdateTimestamp and this will result in only a couple of seconds period for which he'll receive rewards.
## Impact
An attacker can continuously update lastUpdateTimestamp and manipulate the rewards for other legitimate users making them get much less rewards than deserved.
## Tools Used

Manual review

## Recommendations
Keep a separate mapping for the last updated timestamp for each user.

## <a id='M-01'></a>M-01. Wrong check in _requireOnlyOperatorOrOwnerOf in MlumStaking.sol leading to anyone being able to add to someone else's position

## Summary
The function _requireOnlyOperatorOrOwnerOf is intended to check if a userAddress has privileged rights on a spNFT and revert if not.
However, it serves no purpose in the way it's currently implemented.

## Vulnerability Details
Let's see the implementation of _requireOnlyOperatorOrOwnerOf:
```solidity
    /**
     * @dev Check if a userAddress has privileged rights on a spNFT
     */
    function _requireOnlyOperatorOrOwnerOf(uint256 tokenId) internal view {
        // isApprovedOrOwner: caller has no rights on token
        require(ERC721Upgradeable._isAuthorized(msg.sender, msg.sender, tokenId), "FORBIDDEN"); 
    }
```
The function require that ERC721Upgradeable._isAuthorized(msg.sender, msg.sender, tokenId) returns true:
```solidity
    function _isAuthorized(address owner, address spender, uint256 tokenId) internal view virtual returns (bool) {
        return
            spender != address(0) &&
            (owner == spender || isApprovedForAll(owner, spender) || _getApproved(tokenId) == spender);
    }
```
However, since we're passing msg.sender to both the owner and spender parameters, the condition owner == spender will always return true, making the whole function always return true.

The _requireOnlyOperatorOrOwnerOf function is used in  addToPosition() with the intent that it should prevent anyone else but the owner of the staking position or operators adding to the specific position. However, this invariant currently doesn't hold true.
```solidity
    /**
     * @dev Add to an existing staking position
     *
     * Can only be called by lsNFT's owner or operators
     */
    function addToPosition(uint256 tokenId, uint256 amountToAdd) external override nonReentrant {
        
        _requireOnlyOperatorOrOwnerOf(tokenId); <--

       //unrelated

        // we calculate the avg lock time:
        // lock_duration = (remainin_lock_time * staked_amount + amount_to_add * inital_lock_duration) / (staked_amount + amount_to_add)
        uint256 remainingLockTime = _remainingLockTime(position);
        uint256 avgDuration = (remainingLockTime * position.amount + amountToAdd * position.initialLockDuration)
            / (position.amount + amountToAdd);

        position.startLockTime = _currentBlockTimestamp();
        position.lockDuration = avgDuration;

        //unrelated
    }
```
As demonstrated, anyone is able to add to anyone else's staking position. And since the added amount impacts the duration of the lock time, it enables an attacker to perpetually extend someone else's position leaving him in a locked state for as long as he wants.
## Impact
Anyone can add to someone else's position, forcefully extending their locked period.
## Tools Used

Manual review

## Recommendations
Pass the owner in the _requireOnlyOperatorOrOwnerOf, like that
```solidity
    function _requireOnlyOperatorOrOwnerOf(uint256 tokenId) internal view {
        address owner = _ownerOf(tokenId);
        // isApprovedOrOwner: caller has no rights on token
        require(ERC721Upgradeable._isAuthorized(owner, msg.sender, tokenId), "FORBIDDEN"); 
    }
```
## <a id='M-02'></a>M-02. Voter.sol::onRegister() - anyone can register a fake rewarders and DOS the registration of new legit ones

## Summary
Anyone can create a rewarder with a fake ERC20 token that doesn't have any value, then register it Voter.sol::onRegister() and DOS the creation of legit rewarders, essentially making creating bribe rewarders for pools impossible and therefore making the whole bribing system unusable.
## Vulnerability Details
Let's check the createBribeRewarder function in RewarderFactory.sol:
```solidity
    function createBribeRewarder(IERC20 token, address pool) external returns (IBribeRewarder rewarder) {
        rewarder = IBribeRewarder(_cloneBribe(RewarderType.BribeRewarder, token, pool));

        emit BribeRewarderCreated(RewarderType.BribeRewarder, token, pool, rewarder);
    }
```
The bribe rewarder is created and pushed to the array of rewarders without any conditions about what type of ERC20 token is issued as a reward. So the creation of a rewarder with a worthless ERC20 token is possible.

Now let's take a look at the onRegister() function in the Voter.sol contract:
```solidity
    function onRegister() external override { 
        IBribeRewarder rewarder = IBribeRewarder(msg.sender);
        _checkRegisterCaller(rewarder);

        uint256 currentPeriodId = _currentVotingPeriodId;
        (address pool, uint256[] memory periods) = rewarder.getBribePeriods();
        
        for (uint256 i = 0; i < periods.length; ++i) {
            require(periods[i] >= currentPeriodId, "wrong period"); 
           
 //Ensures that the number of bribe rewarders registered for a specific pool in a given period does not exceed a predefined maximum limit
            require(_bribesPerPriod[periods[i]][pool].length + 1 <= Constants.MAX_BRIBES_PER_POOL, "too much bribes");

            _bribesPerPriod[periods[i]][pool].push(rewarder);
        }
    }
```
As you can see, because of this require, there is only a certain amount of bribe rewarders that can be created for a pool:
require(_bribesPerPriod[periods[i]][pool].length + 1 <= Constants.MAX_BRIBES_PER_POOL, "too much bribes");
And in the Constants.sol library that number is defined as 5:
uint256 internal constant MAX_BRIBES_PER_POOL = 5; so there can be a maximum of 5 bribe rewarders per pool before adding new bribe rewarders is disabled.

So what a malicious user would do it:
1. Sees that a pool is launched
2. Creates 5 different bribe rewarders with worthless erc20 tokens and registers them
3. As a result the registration of new legit bribe rewarders is made impossible

This costs basically nothing for the attacker and makes the bribe rewards system completely unusable.
## Impact
An attacker can render the bribe system useless for no cost except some gas.
## Tools Used

Manual review

## Recommendations
I suggest that you implement a whitelist of tokens that can be issued as a reward and check if the token that is used when creating a rewarder via createBribeRewarder is whitelisted. Then when calling fundAndBribe and bribe in BribeRewarder.sol, require that the rewarder is funded with at least some reasonable amount of tokens.

That way, a rewarder can issue only whitelisted tokens, and needs to be funded with reasonable amount of tokens, making the attack expensive and guaranteeing that the users will actually receive rewards that are worth something.






