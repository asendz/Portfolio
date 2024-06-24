# EventShares Security Review

The EventShares smart contract is designed to facilitate a dynamic and engaging way for users to interact with event-based shares, such as Euro 2024.

Users can create and trade shares associated with different teams in the tournament. Only shares associated with the winning team can be converted to ETH, while non-winning shares can be swapped for other shares.

This directs liquidity to the winners, providing novel mechanics for unlimited upside with sports (or any type of) betting.

Author: [0xAsen](https://x.com/asen_sec) - Independent Security Researcher

## Disclaimer
A smart contract security review can never verify a complete absence of vulnerabilities nor it could guarantee finding any of them. 

The only guarantee is that I've spent a sufficient amount of time reviewing and understanding the protocol and did my best to find security breaches and mistakes.

## Risk classification

| Severity           | Impact: High | Impact: Medium | Impact: Low |
| :----------------- | :----------: | :------------: | :---------: |
| Likelihood: High   |   Critical   |      High      |   Medium    |
| Likelihood: Medium |     High     |     Medium     |     Low     |
| Likelihood: Low    |    Medium    |      Low       |     Low     |

### Impact

- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.
- **Low** - can lead to any kind of unexpected behaviour with some of the protocol's functionalities that's not so critical.

### Likelihood

- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no incentive.

# Summary

| ID      | Title                                                                                        | Severity         | Status
| :------ | :------------------------------------------------------------------------------------------- | :--------------- | :--------------- |
| [H-01]  | Users can sell their non-winning shares into ETH| High| Fixed |
| [M-01]  | Incorrect require in `createCoin`| Medium| Fixed |
| [M-02]  | Overpayment of ETH is not handled in `buyShares()`| Medium| Fixed |
| [L-01]  | Single-step ownership transfer is risky| Low | Fixed |
| [L-02]  | `swapShares` is marked as payable but doesn't handle ETH| Low| Fixed |
| [L-03]  | Redundant logic in `createCoin`| Low | Fixed |
| [L-04]  | Users that own too many shareIds won't be able to sell their entire balance| Low | Fixed |                                

# Findings

## [H-01] Users can sell their non-winning shares into ETH 
### Vulnerability detail
The system is designed in a way that it should not allow users to sell their shares in ETH unless the share is from the winning option. 
They are only allowed to swap to other shares(coins) internally. 
This is the require in `sellSharesInternal` that should make sure of that:
```
require(isShareWinning(sharesId) || sellViaSwap, "can only sell winning share or swap into other shares");
```
However, because of the way the `swapShares` function is implemented, this is can be easily circumvented:
```
    function swapShares(string calldata buySharesId, uint256 buyAmount, string calldata sellSharesId, uint256 sellAmount) public payable {
        sellSharesInternal(sellSharesId, sellAmount, true /* sellViaSwap */); 
        buyShares(buySharesId, buyAmount);
    }
```
This scenario can happen:
- There's a room with 2 options - share1 and share2
- User buys a lot of share1. Let's say he makes a good trade and now has 1000$ worth of share1. 
- At this point, the user should not be able to sell his shares into ETH as the option to which the share belongs is not yet declared as winning
- User decides to circumvent this and calls `swapShares` with 1$ worth of `buyAmount` of share2 and his whole balance of share1 as `sellAmount`

The result is that the user will buy only 1$ worth of share2 and sell 999$ worth of share1 into ETH which should not happen and is one of the critical functionalities of the protocol.
### Recommended mitigation steps
- Modify the sellSharesInternal function to return the proceeds after selling the shares.
- Update swapShares function to use the proceeds to buy new shares: Calculate the buyAmount based on the proceeds received from selling the shares and use this amount to buy the new shares.

### Fixes Review

Fixed.

## [M-01] Incorrect require in `createCoin()` 
### Vulnerability detail
When a room is initially created, the options for creating shares is fixed and should not be changed in any circumstances. For example, if a US election room is created initially with coins "Dems" and "Reps", these coins are assigned 0 and 1 option accordingly and users should be able to create coins only under these 2 options

However, with the current implementation, creating another option is possible.

This is the require statement that should make sure that only options with the correct index are created:
```
require(optionIndex < roomToSharesMapping[roomId].length, "invalid option index");
```

However, the require statement compares the option listing to the number of shares(coins) a room has. So this scenario is possible:
- we create a room with coins "Dems" and "Reps" which in turn creates the options 0 and 1 and we should be able to continue creating other coins only under these 2 options
- we make coins "Boden" and "Tremp" so now we have 4 coins in this room
- we can call `createCoin()` for another coin with optionIndex 2 and since the `roomToSharesMapping[roomId].length` will be 4, it'd pass, assigning the new coin under optionIndex 2 making the total options 3

This breaks the invatiant that the options should be fixed at the time of creation of the room and should always stay the same number.
### Recommended mitigation steps
Implement it like this:
```solidity
require(optionIndex < roomToOptionDisplayNameMapping[roomId].length, "invalid option index");
```

### Fixes Review

Fixed.
## [M-02] Overpayment of ETH is not handled in `buyShares()` 
### Vulnerability detail
In the `buyShares()` function users pay with ETH to buy shares. This is the require that makes sure that the user sent enough ETH:
```
require(msg.value >= price + protocolFee, "Insufficient payment");
```
However, if the user overpays, he won't be refunded the overpaid amount. This makes it extremely inconvenient for the user because he'll have to calculate first exactly what amount of shares he wants to buy and send the exact amount of ETH. 
Given the inconvenience and that the prices of the shares could be very volatile and can change at any moment, overpayment is guaranteed to happen.

Which leads to 1) the user paying more than he's supposed to and 2) the ETH gets stuck in the contract because there's no "rescue" function that can be used to retrieve the ETH from the contract.
### Recommended mitigation steps
Add this at the end of the function:
```
if (msg.value > price + protocolFee) {
    uint256 refund = msg.value - (price + protocolFee);
    (bool success, ) = msg.sender.call{value: refund}("");
    require(success);
}
```
### Fixes Review

Fixed.

## [L-01] Single-step ownership transfer is risky

### Vulnerability detail
A single-step process for critical ownership transfer is risky due to possible human error which could result in locking all the functions that use the onlyOwner modifier.

For example, an incorrect address, for which the private key is not known, could be passed accidentally.
### Recommended mitigation steps
[`Ownable2Step`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3d7a93876a2e5e1d7fe29b5a0e96e222afdc4cfa/contracts/access/Ownable2Step.sol#L31-L56) and [`Ownable2StepUpgradeable`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/25aabd286e002a1526c345c8db259d57bdf0ad28/contracts/access/Ownable2StepUpgradeable.sol#L47-L63) prevent the contract ownership from mistakenly being transferred to an address that cannot handle it (e.g. due to a typo in the address), by requiring that the recipient of the owner permissions actively accept via a contract call of its own.
### Fixes Review

Fixed.
## [L-02] `swapShares` is marked as payable but don't handle ETH

### Vulnerability detail
function sellShares(string calldata sharesId, uint256 amount) public payable
```
is marked as payable but it dosn't handle incoming ETH, it is only used to transfer ETH out of the contract. 
### Recommended mitigation steps
Remove the `payable` modifier. 

### Fixes Review

Fixed.
## [L-03] Redundant logic in `createCoin`
### Vulnerability detail
```
    function createCoin(string calldata roomId, uint optionIndex, string calldata sharesId, string calldata symbol) public {
        //ensure the room exists
        require(roomToSharesMapping[roomId].length > 0, "room does not exist");
        //the option under which the coin will be created must exist
        require(optionIndex < roomToSharesMapping[roomId].length, "invalid option index");
        //get the supply of the sharesId
        uint256 supply = sharesSupply[sharesId];
        //supply should be 0
        require(supply == 0, "Shares already created");

        // Check that the share ID is unique within the room @audit this is redundant
        string[] memory optionShares = roomToSharesMapping[roomId];
        for (uint i = 0; i < optionShares.length; i++) {
            require(keccak256(bytes(sharesId)) != keccak256(bytes(optionShares[i])), "Shares already created");
        }
```
There is no need to check if the share ID is unique withing the room since we already guarantee that the share ID is generally unique at this line:
```
require(supply == 0, "Shares already created");
```
### Recommended mitigation steps
Remove this part:
```
        string[] memory optionShares = roomToSharesMapping[roomId];
        for (uint i = 0; i < optionShares.length; i++) {
            require(keccak256(bytes(sharesId)) != keccak256(bytes(optionShares[i])), "Shares already created");
        }
```

### Fixes Review

Fixed.
## [L-04] Users that own too many shareIds won't be able to sell their entire balance
### Vulnerability detail
The following functions are called inside `sellSharesInternal`:
```
    if (sharesBalance[sharesId][msg.sender] == 0) {
            removeSharesIdFromUser(msg.sender, sharesId);
        }
    if (userToRoomSharesMapping[msg.sender][roomId] <= 0) {
            removeUserRoom(msg.sender, roomId);
            removeUserFromRoom(msg.sender, roomId);
        }
```
That loop through each shareId, user in a room, and rooms in which the user participates:
```
function removeSharesIdFromUser(address user, string memory sharesId) internal {
        uint256 length = userToSharesMapping[user].length;
        
        for (uint256 i = 0; i < length; i++) { //@audit looping through all user shares
            if (keccak256(bytes(userToSharesMapping[user][i])) == keccak256(bytes(sharesId))) {
                userToSharesMapping[user][i] = userToSharesMapping[user][length - 1];
                userToSharesMapping[user].pop();
                break;
            }
        }
    }

function removeUserRoom(address user, string memory roomId) internal {

        uint256 length = userRooms[user].length;

        for (uint256 i = 0; i < length; i++) {
            if (keccak256(bytes(userRooms[user][i])) == keccak256(bytes(roomId))) {
                userRooms[user][i] = userRooms[user][length - 1];
                userRooms[user].pop();
                break;
            }
        }
    }
function removeUserFromRoom(address user, string memory roomId) internal {
        uint256 length = roomToUsersMapping[roomId].length;
        
        for (uint256 i = 0; i < length; i++) {
            if (roomToUsersMapping[roomId][i] == user) {
                roomToUsersMapping[roomId][i] = roomToUsersMapping[roomId][length - 1];
                roomToUsersMapping[roomId].pop();
                break;
            }
        }
    }
```
If the number of these becomes too big there's a risk that because of the excessive looping, the user might not be able to sell his shares because of an out-of-gas error.

Low severity because the number of rooms and shareIds must be big and this can be avoided if the user doesn't sell his entire balance and therefore these functions won't be invoked. 
### Recommended mitigation steps
Consider limiting the number of shareIds a user can own and the number of rooms he can enter. 

### Fixes Review

Fixed.
