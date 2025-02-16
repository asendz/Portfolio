# ArkProject: NFT Bridge - Findings Report

## Table of contents
- ### High Risk Findings
    - #### H-01. Users that bridged a token from native L2 collection to L1 can't bridge it back
    - #### H-02. Admin won't be able to remove certain collections from the whitelist on L2
    - #### H-03. If whitelist is off someone can add a lot of collections in the whitelist, DOSing the creation and whitelisting of legitimate collections
- ### Medium Risk Findings
    - #### M-01. A malicious actor can stuck other unsuspecting users' tokens forever
    - #### M-02. User will lock his tokens if he passes `use_withdraw_auto` as true when bridging L2 -> L1



# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-ark-project)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 2
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Users that bridged a token from native L2 collection to L1 can't bridge it back            

## Summary

If you bridge native L2 collection to L1, you can't bridge it back because of a failed if statement in `verify_collection_address`. This check will fail:

```Solidity
       if l2_bridge != l2_req {
            panic!("Invalid collection L2 address");
        }
```

## Vulnerability Details

Let's consider the scenario of having a native L2 collection, not yet bridged. Now let's follow the flow of bridging a token and then bridging it back.\
We call `deposit_tokens()` on the L2 contract, `collection_l2` is the address of the collection on the L2 and this returns address(0) since the l1 -> l2 mappings are not updated because we bridge for the first time:

```Solidity
let collection_l1 = self.l2_to_l1_addresses.read(collection_l2);
```

Now, the message is processed and we call `withdrawTokens()` on the L1 contract. This returns address(0) since the collection is yet to be deployed on the L1:

```Solidity
address collectionL1 = _verifyRequestAddresses(req.collectionL1, req.collectionL2);
```

and we enter this if:

```Solidity
        if (collectionL1 == address(0x0)) {
            //and the type is erc721
            if (ctype == CollectionType.ERC721) {
                //deploy the collection
                collectionL1 = _deployERC721Bridgeable(
                    req.name,
                    req.symbol,
                    req.collectionL2,
                    req.hash
                );
```

which proceeds to deploy the collection and correctly updates the `_l1ToL2Addresses` and `_l2ToL1Addresses` mappings:

```Solidity
    function _deployERC721Bridgeable(
        string memory name,
        string memory symbol,
        snaddress collectionL2,
        uint256 reqHash
    )
        internal
        returns (address)
    {
        address proxy = Deployer.deployERC721Bridgeable(name, symbol);

        _l1ToL2Addresses[proxy] = collectionL2;

        _l2ToL1Addresses[collectionL2] = proxy;

        return proxy;
    }
```

Now let's try and bridge back the same token. We call `depositTokens()` on L1 and these expressions return the stored addresses for the collection both for L1 and L2 since we updated them with the deployment of the collection on L1:

```Solidity
        req.collectionL1 = collectionL1;
        req.collectionL2 = _l1ToL2Addresses[collectionL1];
```

The request is processed and `withdraw_auto_from_l1()` is called on the L2. Let's focus on this line:

```Solidity
let collection_l2 = ensure_erc721_deployment(ref self, @req);
```

and focus on the first lines of the function:

```Solidity
    fn ensure_erc721_deployment(ref self: ContractState, req: @Request) -> ContractAddress {

        //extract the L1 and L2 collection addresses from the request
        let l1_req: EthAddress = *req.collection_l1;
        let l2_req: ContractAddress = *req.collection_l2;

        //verify if the collection address is valid and deployed on L2
        let collection_l2 = verify_collection_address(
            l1_req,
            l2_req,
            self.l2_to_l1_addresses.read(l2_req),
            self.l1_to_l2_addresses.read(l1_req),
        );
```

Here the values for `l1_req` and `l2_req` will be the corresponding addresses of the collection on L1 and L2 since that's what we passed to the request. However, `self.l2_to_l1_addresses.read(l2_req)` and `self.l1_to_l2_addresses.read(l1_req)` both will be address(0) since these mappings were never updated.

Now let's see the `verify_collection_address()` function:

```Solidity
fn verify_collection_address( 
    l1_req: EthAddress,
    l2_req: ContractAddress,
    l1_bridge: EthAddress,
    l2_bridge: ContractAddress,
) -> ContractAddress {

    
    if l1_req.is_zero() {
        panic!("L1 address cannot be 0");
    }


    if l2_req.is_zero() {
        if l2_bridge.is_zero() {
            return ContractAddressZeroable::zero();
        }
   
    } else {
        if l2_bridge != l2_req {
            panic!("Invalid collection L2 address");
        }
        
        if l1_bridge != l1_req {
            panic!("Invalid collection L1 address");
        }
    }

    l2_bridge
}
```

Since `l2_req` is not zero we'll enter the else statement and this will fail:

```Solidity
        if l2_bridge != l2_req {
            panic!("Invalid collection L2 address");
        }
```

because `l2_bridge` represents the address of the L2 collection stored in the bridge contract which currently is 0, while the `l2_req` represents the address we passed with the request, which is not 0.

## Impact

When users bridge a native L2 collection from L2 -> L1 they won't be able to bridge back breaking the most critical functionality of the protocol

## Tools Used

Manual review

## Recommendations

I think it's safe to assume the `l2_req` value is correct since it's the L1 bridge contract that passes it. So you can just return `l2_req` in the else statement.

## <a id='H-02'></a>H-02. Admin won't be able to remove certain collections from the whitelist on L2            



## Summary

Admin won't be able to remove collections from the whitelist on L2 in certain cases. The issue stems from the fact that there's a missing line in the `_white_list_collection` function in the `bridge.cairo` contract that will make the function loop indefinitely if the collection to be removed is after 2nd place in the linked list.

Removing a collection by an admin should always be possible. This will have an even bigger impact if a malicious collection finds its way into the whitelist either by accident or at a time that the whitelist has been off.

## Vulnerability Details

The `white_list_collection()` function is used by admins to add or remove collections from the whitelist. It utilizes the internal `_white_list_collection()`. Let's take a look at it:

```Solidity
fn _white_list_collection(ref self: ContractState, collection: ContractAddress, enabled: bool) {
        let no_value = starknet::contract_address_const::<0>();
  
        let (current, _) = self.white_listed_list.read(collection);
    
        // // If the current status is not the same as the desired status
        if current != enabled {
            let mut prev = self.white_listed_head.read();
            
            //if adding to whitelist
            if enabled {
             //functionality for enabling a collection on the whitelist
            }

            //if disabling the collection
            } else { 

                //if the collection is the head of the list
                if prev == collection {
                    //read the next element in the list from the head
                    let (_, next) = self.white_listed_list.read(prev);
                    //write the current collection as disabled
                    self.white_listed_list.write(collection, (false, no_value));
                    //update the head of the list to the next element
                    self.white_listed_head.write(next);
                    //exit the function since the collection was at the head and is now removed
                    return;
                }

                //if the collection is not the head, remove it from the linked list
                loop {
                    //read the current active status and the next element
                    let (active, next) = self.white_listed_list.read(prev);
                  
                    //if the next element is zero, end of list is reached, break the loop
                    if next.is_zero() {
                        // end of list
                        break;
                    }
                  
                    //if the current element is not active, break the loop
                    if !active {
                        break;
                    }
                 
                    //if the next element is the collection to be removed
                    if next == collection {
                        //read the next element after the collection to be removed
                        let (_, target) = self.white_listed_list.read(collection);
                        //update the previous element to point to the element after the collection
                        self.white_listed_list.write(prev, (active, target));
                        break;
                    }

                //here we should have prev = next; <---
                };

                self.white_listed_list.write(collection, (false, no_value));
            }
        }
    }
```
Let's consider the following scenario:
We have a linked list structure of the following collections: A -> B -> C -> D and we want to remove C

- `let mut prev = self.white_listed_head.read();` sets prev to A.
Entering the else branch since enabled is false and we're removing a collection from the whitelist.
- `if prev == collection` is false since A != C so we go to the loop

First Loop Iteration:
- `let (active, next) = self.white_listed_list.read(prev);` reads B (next of A).
- Since B is not zero and active, these ifs are skipped `if next.is_zero()`, `!active`, and the function proceeds.
- Since next (B) is not the target collection (C), this returns false `if next == collection` and the if is skipped.

At this point, the loop should move to the next element, but the missing line `prev = next`; is not there.
The loop incorrectly processes B again in the next iteration.

This will continue indefinitely since no break condition will be met, causing the function to revert because of out-of-gas at some point and making the target collection impossible to remove.

## Impact
Admins can't remove certain collections from the whitelist breaking the core functionality of the `_white_list_collection` function.
## Tools Used
Manual review
## Recommendations
Add the `prev = next` line at the spot I've pointed out in the code above.
## <a id='H-03'></a>H-03. If whitelist is off someone can add a lot of collections in the whitelist, DOSing the creation and whitelisting of legitimate collections            



## Summary

If whitelist is off, someone can add a lot of collections to the whitelist DOSing the creation and whitelisting of legitimate collections. The DOS would occur in this function in Bridge.sol:

```Solidity
    function _whiteListCollection(address collection, bool enable) internal {
        if (enable && !_whiteList[collection]) {
            bool toAdd = true;
            uint256 i = 0;
            while(i < _collections.length) { <---
                if (collection == _collections[i]) {
                    toAdd = false;
                    break;
                }
                i++;
            }
            if (toAdd) {
                _collections.push(collection);
            }
        }
        _whiteList[collection] = enable;
    }
```

## Vulnerability Details

When the whitelist is off on any of the bridges, anyone can add whatever collection they like by bridging it from L1 -> L2 or vice versa. Let's take a look at this part of the Bridge::withdrawToken() function:

```Solidity
    function withdrawTokens(
        uint256[] calldata request
    )
        external
        payable
        returns (address)
    { 
        if (!_enabled) {
            revert BridgeNotEnabledError();
        }

        uint256 header = request[0];

        if (Protocol.canUseWithdrawAuto(header)) {
            revert NotSupportedYetError();
        } else {
            _consumeMessageStarknet(_starknetCoreAddress, _starklaneL2Address, request);
        }

        Request memory req = Protocol.requestDeserialize(request, 0);

        address collectionL1 = _verifyRequestAddresses(req.collectionL1, req.collectionL2);

        CollectionType ctype = Protocol.collectionTypeFromHeader(header);

        if (collectionL1 == address(0x0)) {
            if (ctype == CollectionType.ERC721) {
                collectionL1 = _deployERC721Bridgeable(
                    req.name,
                    req.symbol,
                    req.collectionL2,
                    req.hash
                );

               //update whitelist
                _whiteListCollection(collectionL1, true);
            } else {
                revert NotSupportedYetError();
            }
        }

```

As you can see when the collection is not deployed it'll be automatically added to the whitelist. Now let's take a closer look at the `_whiteListCollection` function:

```Solidity
    function _whiteListCollection(address collection, bool enable) internal {

        if (enable && !_whiteList[collection]) {
            bool toAdd = true;
            uint256 i = 0;
            
           // loop through all of the collections
            while(i < _collections.length) 
            {
                if (collection == _collections[i]) {
                    toAdd = false;
                    break;
                }
                i++;
            }
            if (toAdd) {
                //push the collection to the array
                _collections.push(collection); 
            }
        }

        _whiteList[collection] = enable; 
    }
```

As you can see, the function loops through all of the collections no matter if they are whitelisted or not.&#x20;

So what could happen is that when the whitelist is off, a malicious actor can create a very big number of collections, in our example on L2, then bridge them to L1, which adds them both to the whitelist and to the collections array, causing the `_whiteListCollection` function to revert due to an Out-of-Gas error.

This will DOS the whitelisting of collections and by extension DOS on the `withdrawTokens()` function which would lead to legitimate users' tokens being stuck permanently in the contract. In our example, the tokens will be stuck in the L2 contract and won't be withdrawable on the L1 contract.

The impact and the possibility of this attack are increased since both sides of the bridge are deployed with the whitelist being off by default:

```Solidity
    bool _whiteListEnabled;

    function initialize(
        bytes calldata data
    )
        public
        onlyInit
    {
        (
            address owner,
            IStarknetMessaging starknetCoreAddress,
            uint256 starklaneL2Address,
            uint256 starklaneL2Selector
        ) = abi.decode(
            data,
            (address, IStarknetMessaging, uint256, uint256)
        );
        _enabled = false;
        _starknetCoreAddress = starknetCoreAddress;

        _transferOwnership(owner);

        setStarklaneL2Address(starklaneL2Address);
        setStarklaneL2Selector(starklaneL2Selector);
    }
```

The `_whiteListEnabled` bool is not set during initialization and bool's default value is `false`. And on the L2 side:

```Solidity
fn constructor(
        ref self: ContractState,
        bridge_admin: ContractAddress,
        bridge_l1_address: EthAddress,
        erc721_bridgeable_class: ClassHash,
    ) {
        self.ownable.initializer(bridge_admin);
        self.bridge_l1_address.write(bridge_l1_address);
        self.erc721_bridgeable_class.write(erc721_bridgeable_class);
        self.white_list_enabled.write(false);

        self.enabled.write(false); // disabled by default <---
    }
```

This opens up the possibility that a malicious actor could perform the attack in between deployment and enabling the whitelist but it's also possible to be executed each time the whitelist is being off.&#x20;

Once the attack is performed the bridge would be unusable by any new collections since there is no mechanism to remove collections from the collections array.

## Impact

Users' tokens will get stuck permanently in the bridge contracts.

## Tools Used

Manual review

## Recommendations

Find a more efficient way to loop through the collections array or remove the array entirely. We're only interested in the whitelisted collection anyway.
If you decide to stick with the collections array, add a function that allows an admin to remove elements from it.
I'd also suggest that the bridges should be deployed with an enabled whitelist by default.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. A malicious actor can stuck other unsuspecting users' tokens forever            



## Summary

A malicious actor can stuck other unsuspecting users' tokens forever.

The problem stems from the fact that there's a reentrancy vulnerability in the Escrow::\_withdrawFromEscrow() function that can be used on any collection to trap unsuspecting users to lock their tokens permanently if they use the bridge.

## Vulnerability Details

The attack can be performed on any collection so let's take Everai for example.

Let's say that I've bridged my NFT with `tokenId` from L1 -> L2 and vice versa. Currently, the token is sitting in the L1 escrow and I have it on L2.

Now here's how the attack would go.

I bridge my token from L2 -> L1 where I specify the `owner_l1` address i.e the `to` address that will receive the NFT on L1 to be an address to a contract I control with the following functionality:

```Solidity
contract Attacker {

   address bridge = address(L1Bridge);

   function withdrawFromBridge(uint256 calldata request) {
       bridge.withdrawTokens(request);
   }
    
   function onERC721Received(address operator, address from, uint256 tokenId, bytes memory data) {
       bridge.depositTokens(someSalt, collectionL1, ownerL2, tokenId); 
    }
}
```

The code above illustrates the basic functionality of the attacker contract.

Now, I'll call the `withdrawFromBridge` from my contract which will call the `withdrawTokens` function on the bridge contract and we'll enter the following functionality:

```Solidity
        for (uint256 i = 0; i < req.tokenIds.length; i++) {
            uint256 id = req.tokenIds[i];

            bool wasEscrowed = _withdrawFromEscrow(ctype, collectionL1, req.ownerL1, id);

            if (!wasEscrowed) {
                IERC721Bridgeable(collectionL1).mintFromBridge(req.ownerL1, id);
            }
```

Let's see what `_withdrawFromEscrow` does:

```Solidity
    function _withdrawFromEscrow(
        CollectionType collectionType,
        address collection,
        address to,
        uint256 id
    )
        internal
        returns (bool)
    {
        if (!_isEscrowed(collection, id)) {
            return false;
        }

        address from = address(this);

        if (collectionType == CollectionType.ERC721) {

            //transfer the token from this contract to `to`
            IERC721(collection).safeTransferFrom(from, to, id);  <--- we reenter from here

        } else {

            IERC1155(collection).safeTransferFrom(from, to, id, 1, "");
        }

        <--- the rest of the function after the reentrancy will be executed from here        

        //set the depositor as address(0)
        _escrow[collection][id] = address(0x0);

        return true;
    }
```

Notice how setting the depositor to address(0) with `_escrow[collection][id] = address(0x0);` is made after the transfer.

Since our token is currently sitting in the escrow on L1, the if is skipped, and the token is transferred with this `IERC721(collection).safeTransferFrom(from, to, id);`

The important thing to know is that `safeTransferFrom` will call the `onERC721Received` function on the receiving address if the address is a contract. Here's the ERC721 implementation:

```Solidity
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data) public virtual {
        transferFrom(from, to, tokenId);
        ERC721Utils.checkOnERC721Received(_msgSender(), from, to, tokenId, data);
    }
```

As you can see, the tokenId is transferred then `checkOnERC721Received` is called and this is the functionality of the function:

```Solidity
 function checkOnERC721Received(
        address operator,
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) internal {
        if (to.code.length > 0) {
     ---> try IERC721Receiver(to).onERC721Received(operator, from, tokenId, data) returns (bytes4 retval) { 
                if (retval != IERC721Receiver.onERC721Received.selector) {
                    // Token rejected
                    revert IERC721Errors.ERC721InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    // non-IERC721Receiver implementer
                    revert IERC721Errors.ERC721InvalidReceiver(to);
                } else {
                    /// @solidity memory-safe-assembly
                    assembly {
                        revert(add(32, reason), mload(reason))
                    }
                }
            }
        }
```

So as confirmed with the code above we established that the token will be transferred and then the `onERC721Received` function will be called on the receiving contract.

Which, in our case, will deposit the token again to the bridge and bridge it from L1 --> L2. During this `_escrow[collection][id]` will be set to our contract's address, however, as soon as the `onERC721Received` is executed i.e the deposit function on the bridge, we'll return to execution on the `_withdrawFromEscrow` function and we'll execute the last line:

```Solidity
_escrow[collection][id] = address(0x0);
```

Let's summarize what we did:

* we had tokenId in L1 escrow, and we owned in on L2
* we bridged it from L1 -> L2, withdrew and with reentering we deposited it again from L1 to L2&#x20;
* execution continues in the `_withdrawFromEscrow` function and sets `_escrow[collection][id] = address(0x0);` although this mapping should point to the depositor(us)

Now the token is in the L1 escrow with depositor set as address(0) and we have the token on L2.

If we were to sell this token to any unsuspecting user and that user tries to bridge it, the token will be locked forever.

Let's see why.

He'll bridge it from L2 -> L1, and will call `withdrawTokens` on L1 which brings us to this functionality again:

```Solidity
        for (uint256 i = 0; i < req.tokenIds.length; i++) {
            uint256 id = req.tokenIds[i];

            bool wasEscrowed = _withdrawFromEscrow(ctype, collectionL1, req.ownerL1, id);

            if (!wasEscrowed) {
                IERC721Bridgeable(collectionL1).mintFromBridge(req.ownerL1, id);
            }
```

When we enter the `_withdrawFromEscrow` function `wasEscrowed` will return false:

```Solidity
    function _withdrawFromEscrow(
        CollectionType collectionType,
        address collection,
        address to,
        uint256 id
    )
        internal
        returns (bool)
    {
        //if not escrowed, return false
        if (!_isEscrowed(collection, id)) {
            return false;
        }

    //rest of the function
```

since the depositor is set to address(0) and this is what the `_isEscrowed` function is checking:

```Solidity
function _isEscrowed(address collection, uint256 id) internal view returns (bool)
    {
        return _escrow[collection][id] > address(0x0);
    }
```

and `withdrawTokens` will continue with this:

```Solidity
            if (!wasEscrowed) {
                IERC721Bridgeable(collectionL1).mintFromBridge(req.ownerL1, id);
            }
```

which will revert since it'll try to mint a `tokenId` that already exists(it's currently in the bridge contract). Or it'll revert if the the collection was a native L1 collection and `mintFromBridge` function doesn't exist.

Now the unsuspecting user's tokens are permanently stuck on L1 and L2, and there's nothing he can do to unlock them.

This attack doesn't cost anything other than some bridging fees and can be done for a big % of the total supply of any collection. For example, if that's done on most of the Everai NFTs it's almost guaranteed that some user will try to bridge his NFT and get his tokens stuck.

## Impact

User's NFTs are lost forever which greatly damages the reputation of the protocol.

## Tools Used

Manual review

## Recommendations

One option is to use `transferFrom` instead of `safeTransferFrom`. However, an even better option is to just move `_escrow[collection][id] = address(0x0);` above in the function before any interactions. Like that:

```Solidity
    function _withdrawFromEscrow(
        CollectionType collectionType,
        address collection,
        address to,
        uint256 id
    )
        internal
        returns (bool)
    {
        if (!_isEscrowed(collection, id)) {
            return false;
        }

        address from = address(this);

        _escrow[collection][id] = address(0x0);

        if (collectionType == CollectionType.ERC721) {
            IERC721(collection).safeTransferFrom(from, to, id); 
        } else {

            IERC1155(collection).safeTransferFrom(from, to, id, 1, ""); 
        }
    
        return true;
    }
```

## <a id='M-02'></a>M-02. User will lock his tokens if he passes `use_withdraw_auto` as true when bridging L2 -> L1            



## Summary

User will lock his tokens if he passes `use_withdraw_auto` as true when bridging L2 -> L1.

## Vulnerability Details

Currently, passing `use_withdraw_auto` is permitted on the L2 `deposit_tokens` function:

```Solidity
        fn deposit_tokens(
            ref self: ContractState,
            salt: felt252,
            collection_l2: ContractAddress,
            owner_l1: EthAddress,
            token_ids: Span<u256>,
            use_withdraw_auto: bool, <--
            use_deposit_burn_auto: bool,
        )
```

There is nothing that stops a user from passing `true` for it and they might be deceived that such functionality exists and that he'll receive his tokens automatically on L1.

However, on L1 auto withdraw is disabled:

```Solidity
    function withdrawTokens(
        uint256[] calldata request
    )
        external
        payable
        returns (address)
    {
        if (!_enabled) {
            revert BridgeNotEnabledError();
        }

        uint256 header = request[0];

        if (Protocol.canUseWithdrawAuto(header)) {
            revert NotSupportedYetError(); <--
```

which will lead to the users token being stuck in the L2 escrow while he won't be able to withdraw it on L1.

With enough users using the protocol, this is almost guaranteed to happen. There's a high chance someone will try this functionality and lock his tokens even if there are disclaimers on the front end.

Currently, there's no reason for this variable to exist at all and the only thing that can come from it being existant is the risk of users accidentally locking their tokens.

## Impact

Users accidentally locking their tokens

## Tools Used

Manual review

## Recommendations

Remove the opportunity for this variable to be passed by the user if such functionality won't be used. If you need it to build the request, automatically pass it as false.





