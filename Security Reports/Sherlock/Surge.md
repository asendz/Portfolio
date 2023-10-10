# What is Surge?

Surge is a hyperstructure lending protocol that allows anyone to deploy their own lending pool. It utilizes a novel mechanism called Algorithmic Collateral Ratios in order to eliminate the need for price oracles in DeFi lending.

[Link to the contest](https://audits.sherlock.xyz/contests/51)

# Issues found by me

| Severity | Title                                                                                        | Link                                                                       |
| :------- | :------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------- |
| High     | \_loanTokenBalance can be easily manipulated leading to wrong calculations and loss of funds | [Link](https://github.com/sherlock-audit/2023-02-surge-judging/issues/179) |

# 1. \_loanTokenBalance can be easily manipulated leading to wrong calculations and loss of funds

## Summary

\_loanTokenBalance can be easily manipulated leading to wrong calculations and potential loss of funds for users of the pool.

## Vulnerability Detail

\_loanTokenBalance is used for calculation of the shares a user should receive in both the deposit() and the withdraw() functions:

```solidity
function deposit(uint amount) external {
        uint _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this));
        ...
        uint _shares = tokenToShares(amount, (_currentTotalDebt + _loanTokenBalance), _currentTotalSupply, false);
        require(_shares > 0, "Pool: 0 shares");
        _currentTotalSupply += _shares;
				...
}
```

```solidity
function withdraw(uint amount) external {
        uint _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this));
        ...
        uint _shares;
        if (amount == type(uint).max) {
            amount = balanceOf[msg.sender] * (_currentTotalDebt + _loanTokenBalance) / _currentTotalSupply;
            _shares = balanceOf[msg.sender];
        } else {
            _shares = tokenToShares(amount, (_currentTotalDebt + _loanTokenBalance), _currentTotalSupply, true);
        }
        _currentTotalSupply -= _shares;
				...
}
```

However, the \_loanTokenBalance can be easily manipulated by an outside malicious user by just sending as little as 1 loan token to the pool contract leading to wrong calculations of the shares and totalSupply.

The wrong calculation of shares leads to unsuspecting users depositing X tokens but receiving < X shares, making them unable to withdraw the full amount of their deposit later.

The attack is also incredibly easy and cheap to perform(can be done with just 1 token).

## Impact

Loss of funds for all users depositing in the pool after the attacker messed up the calculations by sending loanToken to the pool contract.

## Code Snippet

https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L324

https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L370

## Recommendation

Recommendation
Use another way to keep track of the loan tokens in the pool.

For example, you can keep track of the deposited loan tokens and use that for the calculations. That way, \_loanTokenBalance will keep track only of the loan tokens which are added via the deposit() function and thus will be correctly accounted for.
