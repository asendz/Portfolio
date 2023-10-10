# What is Ondo Finance?
The scope of this audit was `rUSDY` - interest bearing stablecoin, and can be thought of as a rebasing variant of Ondo's USDY (Ondo U.S. Dollar Yield) token. Individuals who hold USDY are able to wrap their USDY tokens and receive an amount of rUSDY tokens proportional to the USD value wrapped. Each rUSDY token is worth 1 dollar, as USDY accrues value (appreciates in price) rUSDY will rebase accordingly.

Where USDY is a tokenized note secured by short-term US Treasuries and bank demand deposits

[Link to the contest](https://code4rena.com/contests/2023-09-ondo-finance)

# Issues found by me

| Severity | Title                                                                                        | Link                                                                       |
| :------- | :------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------- |
| Medium     | Admin can't burn tokens from blocklisted addresses because of a check in `_beforeTokenTransfer` | [Link]() |

# 1. Admin can't burn tokens from blocklisted addresses because of a check in `_beforeTokenTransfer`

## Impact
The function `burn` is made so the admin can burn rUSDY tokens from ***any account*** - this is stated in the comments. However, the admin can't burn tokens if the account from which he's trying to burn tokens is blocklisted/sanctioned/not on the allowlist.
## Proof of Concept
Let's check the `burn` function which calls the internal `_burnShares` function:
```javascript
function burn(
  address _account,
  uint256 _amount
) external onlyRole(BURNER_ROLE) {
  uint256 sharesAmount = getSharesByRUSDY(_amount);

  _burnShares(_account, sharesAmount);

  usdy.transfer(msg.sender, sharesAmount / BPS_DENOMINATOR);

  emit TokensBurnt(_account, _amount);
}

function _burnShares(
  address _account,
  uint256 _sharesAmount
) internal whenNotPaused returns (uint256) {
  require(_account != address(0), "BURN_FROM_THE_ZERO_ADDRESS");

  _beforeTokenTransfer(_account, address(0), _sharesAmount); <--

  uint256 accountShares = shares[_account];
  require(_sharesAmount <= accountShares, "BURN_AMOUNT_EXCEEDS_BALANCE");

  uint256 preRebaseTokenAmount = getRUSDYByShares(_sharesAmount);

  totalShares -= _sharesAmount;

  shares[_account] = accountShares - _sharesAmount;

  uint256 postRebaseTokenAmount = getRUSDYByShares(_sharesAmount);

  return totalShares;
}

```
We can see that it calls `_beforeTokenTransfer(_account, address(0), _sharesAmount)`
Here is the code of `_beforeTokenTransfer`:
```javascript
function _beforeTokenTransfer(
  address from,
  address to,
  uint256
) internal view {
  // Check constraints when `transferFrom` is called to facliitate
  // a transfer between two parties that are not `from` or `to`.
  if (from != msg.sender && to != msg.sender) {
    require(!_isBlocked(msg.sender), "rUSDY: 'sender' address blocked");
    require(!_isSanctioned(msg.sender), "rUSDY: 'sender' address sanctioned");
    require(
      _isAllowed(msg.sender),
      "rUSDY: 'sender' address not on allowlist"
    );
  }

  if (from != address(0)) { <--
    // If not minting
    require(!_isBlocked(from), "rUSDY: 'from' address blocked");
    require(!_isSanctioned(from), "rUSDY: 'from' address sanctioned");
    require(_isAllowed(from), "rUSDY: 'from' address not on allowlist");
  }

  if (to != address(0)) {
    // If not burning
    require(!_isBlocked(to), "rUSDY: 'to' address blocked");
    require(!_isSanctioned(to), "rUSDY: 'to' address sanctioned");
    require(_isAllowed(to), "rUSDY: 'to' address not on allowlist");
  }
}
```
In our case, the `from` would be the account from which we're burning tokens so it'll enter in the 2nd if - `if (from != address(0))`. But given that the account is blocked/sanctioned/not on the allowlist, the transaction will revert and the tokens won't be burned.

Given that there are separate roles for burning and managing the block/sanctions/allowed lists - `BURNER_ROLE` and `LIST_CONFIGURER_ROLE`, it is very possible that such a scenario would occur.

In that case, the Burner would have to ask the List Configurer to update the lists, so the Burner can burn the tokens, and then the List Configurer should update the lists again. However, in that case, you're risking that the blocked user manages to transfer their funds while performing these operations.
## Tools Used
Manual review
## Recommended Mitigation Steps
Organize the logic of the function better. For example, you can make the 2nd if to be:
`if (from != address(0) && to != address(0))`, that way we'll not enter the if when burning tokens, and we'll be able to burn tokens from blocked accounts.
