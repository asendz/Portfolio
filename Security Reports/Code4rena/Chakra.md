# Chakra: Modular settlement layer - Findings Report

## Table of contents
- ### High Risk Findings
    - #### H-01. In settlement.cairo::receive_cross_chain_msg - the payload_type can be passed by the user, confusing offchain systems
    - #### H-02. settlement.cairo doesn't process callback correctly leading to CrossChainMsgStatus marked as SUCCESS even if it failed on destination chain
    - #### H-03. In settlement.cairo::receive_cross_chain_msg - the message will always be marked with Status::SUCCESS
    - #### H-04. When processing a callback, a malicious actor can force mark a message as Failed even if the message was successful
    - #### H-05. handler.receive_cross_chain_callback doesn't process transaction status correctly, always marking it as SETTLED even if the transaction failed on destination chain
    - #### H-06. Status of cross_chain_msg_id not checked in settlement.cairo::receive_cross_chain_msg making replay attacks possible
- ### Medium Risk Findings
    - #### M-01. Users can create 0 amount token transfers, breaking a key invariant of the protocol
    - #### M-02. ChakraSettlementHandler - to_handler is not checked if it's whitelisted

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://code4rena.com/audits/2024-08-chakra)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 6
- Medium: 2


# High Risk Findings

## <a id='H-01'></a>H-01. In settlement.cairo::receive_cross_chain_msg - the payload_type can be passed by the user, confusing offchain systems
## Impact
In `settlement.cairo::receive_cross_chain_msg` - the `payload_type` can be passed by the user, confusing offchain systems.

The `payload_type` parameter is only used to emit events so that the Chakra nodes can detect and process them.

There is no validation for it and given that it is used in an event to which off-chain systems listen to, the `payload_type` values will be displayed in the explorer, and there may be other extensions in the future, according to the sponsor.

This can lead to incorrect data being displayed and undefined behavior in the future.

## Proof of Concept
Let's see the code of `receive_cross_chain_msg`:
```
  fn receive_cross_chain_msg(
      ref self: ContractState,
      cross_chain_msg_id: u256,
      from_chain: felt252,
      to_chain: felt252,
      from_handler: u256,
      to_handler: ContractAddress,
      sign_type: u8,
      signatures: Array<(felt252, felt252, bool)>,
      payload: Array<u8>,
      payload_type: u8, <---
  ) -> bool {
      assert(to_chain == self.chain_name.read(), 'error to_chain');

      // verify signatures
      let mut message_hash: felt252 = LegacyHash::hash(from_chain, (cross_chain_msg_id, to_chain, from_handler, to_handler));
      let payload_span = payload.span();
      let mut i = 0;
      loop {
          if i > payload_span.len()-1{
              break;
          }
          message_hash = LegacyHash::hash(message_hash, * payload_span.at(i));
          i += 1;
      };
      self.check_chakra_signatures(message_hash, signatures);

      // call handler receive_cross_chain_msg
      let handler = IHandlerDispatcher{contract_address: to_handler};
      let success = handler.receive_cross_chain_msg(cross_chain_msg_id, from_chain, to_chain, from_handler, to_handler , payload);

      let mut status = CrossChainMsgStatus::SUCCESS;
      if success{
          status = CrossChainMsgStatus::SUCCESS;
      }else{
          status = CrossChainMsgStatus::FAILED;
      }

      self.received_tx.write(cross_chain_msg_id, ReceivedTx{
          tx_id:cross_chain_msg_id,
          from_chain: from_chain,
          from_handler: from_handler,
          to_chain: to_chain,
          to_handler: to_handler,
          tx_status: status
      });

      // emit event
      self.emit(CrossChainHandleResult{
          cross_chain_settlement_id: cross_chain_msg_id,
          from_chain: to_chain,
          from_handler: to_handler,
          to_chain: from_chain,
          to_handler: from_handler,
          cross_chain_msg_status: status,
          payload_type: payload_type <---
      });
      return true;
  }
```
You can see that the `payload_type` parameter is not used anywhere in the function except for the emission of the event.

So what could happen is:
- a user or a bot sees the transaction being processed
- calls the `receive_cross_chain_msg` function before the Chakra off-chain system with all the right parameters except for the `payload_type`
- since the `payload_type` is not used in the message hash generation, the signatures verification and every other check passes successfully
- the off-chain systems pick up wrong information from the event leading to corrupted information
## Tools Used
Manual review
## Recommended Mitigation Steps
Include the `payload_type` in the message hash generation thus making sure that it's value cannot be altered

## <a id='H-02'></a>H-02. settlement.cairo doesn't process callback correctly leading to CrossChainMsgStatus marked as SUCCESS even if it failed on destination chain
## Impact
`settlement.cairo` doesn't process callback correctly, leading to `CrossChainMsgStatus` marked as `SUCCESS` even if it failed on the destination chain.

## Proof of Concept
When a cross-chain message is sent it can return a callback with status `FAILED` or `SUCCESS`. The problem is that even if the cross-chain message failed, the original message status on the source chain would be marked as `SUCCESS`.

Let's take a look at the `receive_cross_chain_callback` function on the `settlement.cairo` contract, and more specifically the part that updates the status:
```
fn receive_cross_chain_callback(
        ref self: ContractState,
        cross_chain_msg_id: felt252,
        from_chain: felt252,
        to_chain: felt252,
        from_handler: u256,
        to_handler: ContractAddress,
        cross_chain_msg_status: u8, <--
        sign_type: u8,
        signatures: Array<(felt252, felt252, bool)>,
    ) -> bool {
//other functionality

let success = handler.receive_cross_chain_callback(cross_chain_msg_id, from_chain, to_chain, from_handler, to_handler , cross_chain_msg_status);

        let mut state = CrossChainMsgStatus::PENDING;
        if success{
            state = CrossChainMsgStatus::SUCCESS;
        }else{
            state = CrossChainMsgStatus::FAILED;
        }

        self.created_tx.write(cross_chain_msg_id, CreatedTx{
            tx_id:cross_chain_msg_id,
            tx_status:state, <--- update the status
            from_chain: to_chain,
            to_chain: from_chain,
            from_handler: to_handler,
            to_handler: from_handler
        });
```
The problem is that as long as the call to `handler.receive_cross_chain_callback` function was successful, the message as a whole will be marked in a `SUCCESS` state even though that `cross_chain_msg_status` could be SUCCESS or FAILED depending on if the message failed on the destination chain.

This could lead to a situation where a message fails to get processed on the destination chain, a callback is returned with `cross_chain_msg_status == FAILED` but the message is marked as `SUCCESS`.

That situation could be a user trying to bridge his tokens, the bridging fails so he doesn't receive his tokens on the destination chain, a callback is made and the message is marked as a `SUCCESS` even though it as not successfully executed.

And if we see the code of the `handler.receive_cross_chain_callback` function, we'll see that it'd always return true as long as it doesn't revert:
```
    fn receive_cross_chain_callback(
        ref self: ContractState,
        cross_chain_msg_id: felt252,
        from_chain: felt252,
        to_chain: felt252,
        from_handler: u256,
        to_handler: ContractAddress,
        cross_chain_msg_status: u8
    ) -> bool{

        assert(to_handler == get_contract_address(),'error to_handler');
        assert(self.settlement_address.read() == get_caller_address(), 'not settlement');
        assert(self.support_handler.read((from_chain, from_handler)) &&
                self.support_handler.read((to_chain, contract_address_to_u256(to_handler))), 'not support handler');

        let erc20 = IERC20MintDispatcher{contract_address: self.token_address.read()};

        if self.mode.read() == SettlementMode::MintBurn{
            erc20.burn_from(get_contract_address(), self.created_tx.read(cross_chain_msg_id).amount);
        }

        let created_tx = self.created_tx.read(cross_chain_msg_id);

        self.created_tx.write(cross_chain_msg_id, CreatedCrossChainTx{
            tx_id: created_tx.tx_id,
            from_chain: created_tx.from_chain,
            to_chain: created_tx.to_chain,
            from:created_tx.from,
            to:created_tx.to,
            from_token: created_tx.from_token,
            to_token: created_tx.to_token,
            amount: created_tx.amount,
            tx_status: CrossChainTxStatus::SETTLED
        });

        return true;
    }
```
The function just performs validation of the handlers, if the settlement contract calls it and returns true. These validation could all be true but the initial message could still be with a FAILED status.

This is not taken into account and could lead to a failed message being marked as a successful one.

The implementation on the solidity contract is correct:
```
function processCrossChainCallback(
    uint256 txid,
    string memory from_chain,
    uint256 from_handler,
    address to_handler,
    CrossChainMsgStatus status,
    uint8 sign_type,
    bytes calldata signatures
) internal {
    require(
        create_cross_txs[txid].status == CrossChainMsgStatus.Pending,
        "Invalid transaction status"
    );

    if (
        ISettlementHandler(to_handler).receive_cross_chain_callback(
            txid,
            from_chain,
            from_handler,
            status,
            sign_type,
            signatures
        )
    ) {
        create_cross_txs[txid].status = status; <---

    } else {
        create_cross_txs[txid].status = CrossChainMsgStatus.Failed;
    }
}
```
As you can see, even if the call to the handler was successful, the status is updated with the `CrossChainMsgStatus` that was passed to the function and it is not automatically marked with `SUCCESS`.

This should be the case in the cairo function as well but right now the `cross_chain_msg_status` parameter is ignored.
## Tools Used
Manual review
## Recommended Mitigation Steps
Change this line from this:
```
        if success{
            state = CrossChainMsgStatus::SUCCESS;
```
to this:
```
        if success{
            state = cross_chain_msg_status;
```
To be consistent with the solidity implementation.


## <a id='H-03'></a>H-03. In settlement.cairo::receive_cross_chain_msg - the message will always be marked with Status::SUCCESS
## Impact
When receiving a message via the `settlement.cairo::receive_cross_chain_msg()` function, a call is made to `handler.receive_cross_chain_msg` and the return value of that call is used to mark the message with status `SUCCESS` or `FAILED`

However, the way that the `handler.receive_cross_chain_msg` function is implemented, it'll always return true, therefore marking every message as successful even if the message for some reason was a corrupted one.

And breaking one of the main invariants that states the message statuses should be tracked correctly.
## Proof of Concept
Let's take a look at the code of the `receive_cross_chain_msg` function on the settlement.cairo contract, and specifically at the part where the call to `handler.receive_cross_chain_msg` is made:
```
fn receive_cross_chain_msg(/*params*/) -> bool {
//rest of the code

let success = handler.receive_cross_chain_msg(
            cross_chain_msg_id,
            from_chain, //from_chain
            to_chain,  //to_chain
            from_handler, //from_handler
            to_handler , //to_handler
            payload     //payload
        );

        let mut status = CrossChainMsgStatus::SUCCESS;

        if success{
            status = CrossChainMsgStatus::SUCCESS;
        }else{
            status = CrossChainMsgStatus::FAILED;
        }

}
```
As you can see, the status of the messages is based on the return value of this line: `success = handler.receive_cross_chain_msg`.

Let's see the `handler.receive_cross_chain_msg` function:
```
    fn receive_cross_chain_msg(
        ref self: ContractState,
        cross_chain_msg_id: u256,
        from_chain: felt252,
        to_chain: felt252,
        from_handler: u256,
        to_handler: ContractAddress,
        payload: Array<u8>
        ) -> bool{

        assert(to_handler == get_contract_address(),'error to_handler');

        assert(self.settlement_address.read() == get_caller_address(), 'not settlement');

        assert(self.support_handler.read((from_chain, from_handler)) &&
                self.support_handler.read((to_chain, contract_address_to_u256(to_handler))),
                'not support handler');
                //i.e (from_chain, to_handler)

        let message :Message= decode_message(payload);
        let payload_type = message.payload_type;

        assert(payload_type == PayloadType::ERC20, 'payload type not erc20');

        let payload_transfer = message.payload;

        let transfer = decode_transfer(payload_transfer);

        assert(transfer.method_id == ERC20Method::TRANSFER, 'ERC20Method must TRANSFER');

        let erc20 = IERC20MintDispatcher{contract_address: self.token_address.read()};
        let token = IERC20Dispatcher{contract_address: self.token_address.read()};

        // Handle the cross-chain transfer according to the settlement mode set in the contract.

        if self.mode.read() == SettlementMode::MintBurn{
            erc20.mint_to(u256_to_contract_address(transfer.to), transfer.amount);

        }else if self.mode.read() == SettlementMode::LockMint{
            erc20.mint_to(u256_to_contract_address(transfer.to), transfer.amount);

        }else if self.mode.read() == SettlementMode::BurnUnlock{
            token.transfer(u256_to_contract_address(transfer.to), transfer.amount);

        }else if self.mode.read() == SettlementMode::LockUnlock{
            token.transfer(u256_to_contract_address(transfer.to), transfer.amount);
        }

        return true;
    }
```
As you can see, as long as the function doesn't revert, it'll always return `true`.

This shouldn't be the case and if for some reason the message is a corrupted one, it should get marked as `FAILED` as intended by the protocol.

The functionality is correctly implemented in the solidity version:
```
function receive_cross_chain_msg(
    uint256 /**txid */,
    string memory from_chain,
    uint256 /**from_address */,
    uint256 from_handler,
    PayloadType payload_type,
    bytes calldata payload,
    uint8 /**sign type */,
    bytes calldata /**signaturs */
) external onlySettlement returns (bool) {

    if (is_valid_handler(from_chain, from_handler) == false) {
        return false;
    }

    bytes calldata msg_payload = MessageV1Codec.payload(payload);
    require(isValidPayloadType(payload_type), "Invalid payload type");

    if (payload_type == PayloadType.ERC20) {
        {
            ERC20TransferPayload memory transfer_payload = codec
                .deocde_transfer(msg_payload);

            if (mode == SettlementMode.MintBurn) {
                _erc20_mint(
                    AddressCast.to_address(transfer_payload.to),
                    transfer_payload.amount
                );

                return true;
            } else if (mode == SettlementMode.LockUnlock) {
                _erc20_unlock(
                    AddressCast.to_address(transfer_payload.to),
                    transfer_payload.amount
                );

                return true;
            } else if (mode == SettlementMode.LockMint) {

                _erc20_mint(
                    AddressCast.to_address(transfer_payload.to),
                    transfer_payload.amount
                );
                return true;
            } else if (mode == SettlementMode.BurnUnlock) {
                _erc20_unlock(
                    AddressCast.to_address(transfer_payload.to),
                    transfer_payload.amount
                );
                return true;
            }
        }
    }

    return false;
}
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Implement the functionality like in the solidity handler function so it returns `false` if something goes wrong.

## <a id='H-04'></a>H-04. When processing a callback, a malicious actor can force mark a message as Failed even if the message was successful
## Impact
In `ChakraSettlement::receive_cross_chain_callback`, when processing a callback, a malicious actor can force a message to be marked as `Failed` even if the message was successful.

This would result in actions like burning the tokens if the mode is `MintBurn` not being processed and it breaks a key invariant of the protocol that states the message statuses are correctly tracked.
## Proof of Concept
The issue is that the `from_chain` parameter does not take part in the creation of `message_hash` when receiving a callback.

This allows a malicious actor to front-run the legitimate transaction with all the right parameters except for the `from_chain` and process the message with status "Failed" even if the transaction on the destination chain was successfully executed.

Let's take a look at the code of the `receive_cross_chain_callback` function:
```
function receive_cross_chain_callback(
    uint256 txid,
    string memory from_chain, <--
    uint256 from_handler,
    address to_handler,
    CrossChainMsgStatus status,
    uint8 sign_type,
    bytes calldata signatures
) external {

    verifySignature(
        txid,
        from_handler,
        to_handler,
        status,
        sign_type,
        signatures
    );

    processCrossChainCallback(
        txid,
        from_chain,
        from_handler,
        to_handler,
        status,
        sign_type,
        signatures
    );

    emitCrossChainResult(txid);
}
```
As you can see, the function is external so anyone can call it. Let's now check the `verifySignature` function:
```
function verifySignature(
    uint256 txid,
    uint256 from_handler,
    address to_handler,
    CrossChainMsgStatus status,
    uint8 sign_type,
    bytes calldata signatures
) internal view {
    bytes32 message_hash = keccak256(
        abi.encodePacked(
            txid,
            from_handler,
            to_handler,
            status)
    );

    require(
        signature_verifier.verify(message_hash, signatures, sign_type),
        "Invalid signature"
    );
}
```
The function generates the `message_hash` using only `txid`, `from_handler`, `to_handler`, and `status`.

Therefore, if these parameters are the parameters of a legitimate transaction, the `message_hash` will be correctly generated and the signatures will be verified to be valid.

Now let's see the `processCrossChainCallback` function:
```
function processCrossChainCallback(
    uint256 txid,
    string memory from_chain,
    uint256 from_handler,
    address to_handler,
    CrossChainMsgStatus status,
    uint8 sign_type,
    bytes calldata signatures
) internal {
    require(
        create_cross_txs[txid].status == CrossChainMsgStatus.Pending,
        "Invalid transaction status"
    );

    if (
        ISettlementHandler(to_handler).receive_cross_chain_callback(
            txid,
            from_chain,
            from_handler,
            status,
            sign_type,
            signatures
        )
    ) {
        create_cross_txs[txid].status = status;
    } else {
        create_cross_txs[txid].status = CrossChainMsgStatus.Failed;
    }
}
```
If the call to ` ISettlementHandler(to_handler).receive_cross_chain_callback` returns `false`, then `create_cross_txs[txid].status = CrossChainMsgStatus.Failed`

Let's see the `ISettlementHandler(to_handler).receive_cross_chain_callback`:
```
function receive_cross_chain_callback(
    uint256 txid,
    string memory from_chain,
    uint256 from_handler,
    CrossChainMsgStatus status,
    uint8 /* sign_type */, // validators signature type /  multisig or bls sr25519
    bytes calldata /* signatures */
) external onlySettlement returns (bool) {

    if (is_valid_handler(from_chain, from_handler) == false) {
        return false;
    }

//rest of the function
```
Now, if `from_chain` is a made-up chain name, then `is_valid_handler(from_chain, from_handler)` will return false since the `from_handler` won't be whitelisted for our made-up chain name and the `create_cross_txs[txid].status` will be marked as `Failed` in the Settlement contract even if the transaction was successful on the destination chain.

Now let's complete the scenario on how the attack could occur:
- There's a cross-chain transaction that was successfully executed on the destination chain
- A callback transaction is submitted in the mempool by the Chakra system(or keepers, whatever handles their callbacks) for the `ChakraSettlement::receive_cross_chain_callback` function
- A malicious actor sees that call and submits a call on his own copying all of the parameters of the legitimate call except for the `from_chain`. For the `from_chain` parameter, a made-up value is passed. He submits his transaction with higher gas fee, frontrunning the legitimate transaction.
- The malicious actor's transaction gets processed first and the message is marked as `Failed` because the handler is not whitelisted for the made-up `from_chain`. The signatures verification and all of the other checks pass because all of the other parameters are correct as they are copied from the legitimate transaction.
- The system's legitimate transaction invokes the `receive_cross_chain_callback` function but it reverts because the status of the message is not `Pending`:
```
    require(
        create_cross_txs[txid].status == CrossChainMsgStatus.Pending,
        "Invalid transaction status"
    );
```

The result is that a transaction that was succesfully executed on the destination chain is marked as `Failed` on the source chain, and following actions like burning tokens if the mode is `MintBurn` are not executed.

This is completely possible on a chain like Ethereum for example which is in scope.
## Tools Used
Manual review
## Recommended Mitigation Steps
Include the `from_chain` in the generation of the `message_hash`.

Or verify that the `from_chain` i.e the destination chain of the original message(since the callback is initiated from there) is equal to the `to_chain` of the original message.

## <a id='H-05'></a>H-05. handler.receive_cross_chain_callback doesn't process transaction status correctly, always marking it as SETTLED even if the transaction failed on destination chain
## Impact
`handler.receive_cross_chain_callback` doesn't process transaction status correctly, always marking it as SETTLED even if the transaction failed on the destination chain.

This leads to users' transactions getting incorrectly marked and could lead to user funds being lost if the tokens were unsuccessfully bridged to another chain, but burned on the source chain.

It also breaks one of the main invariants of the protocol stated in the ReadMe:
- Cross-chain transaction statuses are properly tracked and updated (Pending, Settled, or Failed).

## Proof of Concept
Let's take a look at the `receive_cross_chain_callback` function in the `handler_erc20.cairo` contract:

```
fn receive_cross_chain_callback(ref self: ContractState, cross_chain_msg_id: felt252, from_chain: felt252, to_chain: felt252,
    from_handler: u256, to_handler: ContractAddress, cross_chain_msg_status: u8) -> bool{
        assert(to_handler == get_contract_address(),'error to_handler');

        assert(self.settlement_address.read() == get_caller_address(), 'not settlement');

        assert(self.support_handler.read((from_chain, from_handler)) &&
                self.support_handler.read((to_chain, contract_address_to_u256(to_handler))), 'not support handler');

        let erc20 = IERC20MintDispatcher{contract_address: self.token_address.read()};
        if self.mode.read() == SettlementMode::MintBurn{
            erc20.burn_from(get_contract_address(), self.created_tx.read(cross_chain_msg_id).amount);
        }

        let created_tx = self.created_tx.read(cross_chain_msg_id);

        self.created_tx.write(cross_chain_msg_id, CreatedCrossChainTx{
            tx_id: created_tx.tx_id,
            from_chain: created_tx.from_chain,
            to_chain: created_tx.to_chain,
            from:created_tx.from,
            to:created_tx.to,
            from_token: created_tx.from_token,
            to_token: created_tx.to_token,
            amount: created_tx.amount,
            tx_status: CrossChainTxStatus::SETTLED <-- marked as SETTLED
        });

        return true;
    }
```
As you can see, the transaction is always marked as `SETTLED` even though the `cross_chain_msg_status` parameter passed to the function could very well be `FAILED` if the transaction on the destination chain was unsuccessful.

This is incorrect and it can be confirmed by the logic in the solidity version of the same function, where a failed transaction is correctly updated with the appropriate status:
```
function receive_cross_chain_callback(
    uint256 txid,
    string memory from_chain,
    uint256 from_handler,
    CrossChainMsgStatus status,
    uint8 /* sign_type */, // validators signature type /  multisig or bls sr25519
    bytes calldata /* signatures */
) external onlySettlement returns (bool) {
    if (is_valid_handler(from_chain, from_handler) == false) {
        return false;
    }

    require(
        create_cross_txs[txid].status == CrossChainTxStatus.Pending,
        "invalid CrossChainTxStatus"
    );

    //if status is Success
    if (status == CrossChainMsgStatus.Success) {
        //and the mode is MintBurn
        if (mode == SettlementMode.MintBurn) {
            _erc20_burn(address(this), create_cross_txs[txid].amount);
        }

        //set the transaction status as settled
        create_cross_txs[txid].status = CrossChainTxStatus.Settled; <--
    }

    //if the status is Failed
    if (status == CrossChainMsgStatus.Failed) {
        //mark the tx status as failed
        create_cross_txs[txid].status = CrossChainTxStatus.Failed; <--
    }

    return true;
}
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Implement if/else logic, like in the solidity version, that marks the transaction as Settled/Failed based on the `cross_chain_msg_status` variable that is currently ignored.

## <a id='H-06'></a>H-06. Status of cross_chain_msg_id not checked in settlement.cairo::receive_cross_chain_msg making replay attacks possible
## Impact
The status of `cross_chain_msg_id` is not checked in `settlement.cairo::receive_cross_chain_msg` making replay attacks possible.

Checking for the status of the message is crucial because otherwise, one message can be processed multiple times which can very well lead to the whole protocol getting drained.

## Proof of Concept
Let's say that I've initiated a cross-chain settlement including ERC20 token transfer, it got executed, and I already received my tokens on Starknet using the settlement and handler contracts.

Now, nothing is stopping me from calling the `receive_cross_chain_msg` function on the `settlement.cairo` contract again with the same data, receiving double the tokens. This can be done repeatedly until the protocol runs out of funds.

As you can see the function is public:
```
fn receive_cross_chain_msg(
      ref self: ContractState,
      cross_chain_msg_id: u256,
      from_chain: felt252,
      to_chain: felt252,
      from_handler: u256,
      to_handler: ContractAddress,
      sign_type: u8,
      signatures: Array<(felt252, felt252, bool)>,
      payload: Array<u8>,
      payload_type: u8,
  ) -> bool {
```
and the signature verification checks will pass since this was a legit message signed by legit validators:
```
      self.check_chakra_signatures(message_hash, signatures);
```
What is missing here is a check that verifies the message is yet to be processed i.e it's not yet marked with status `CrossChainMsgStatus::SUCCESS` or `CrossChainMsgStatus::FAILED`.

The check is present in the solidity version of the contract but it's missing on cairo:
```
      require(
          receive_cross_txs[txid].status == CrossChainMsgStatus.Unknow,
          "Invalid transaction status"
      );
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Implement the same check like in the solidity version, making sure that the message can't be processed more than once.

# Medium Risk Findings

## <a id='M-01'></a>M-01. Users can create 0 amount token transfers, breaking a key invariant of the protocol
## Impact
Users can initiate a 0 amount token transfers in `handler_erc20.cairo::cross_chain_erc20_settlement`

This breaks a main invariant of the protocol stated in the contest description:
- Cross-chain ERC20 settlements can only be initiated with a valid amount (greater than 0), a valid recipient address, a valid handler address, and a valid token address.

It also opens up the possiblity of creating a large amount of cross-chain settlements potentially overwhelming the system with no cost because of 0 amount tokens transfers.
## Proof of Concept
There is no such checks in the cairo `receive_cross_chain_msg` function.

This breaks a main invariant of the protocol that is correctly enforced in the solidity version:
```
  require(amount > 0, "Amount must be greater than 0");
  require(to != 0, "Invalid to address");
  require(to_handler != 0, "Invalid to handler address");
  require(to_token != 0, "Invalid to token address");
```
which makes it even more apparent that these checks should be enforced.
## Tools Used
Manual review
## Recommended Mitigation Steps
Implement the checks in the cairo handler just like in the solidity one.

## <a id='M-02'></a>M-02. ChakraSettlementHandler - to_handler is not checked if it's whitelisted
## Impact
When initiating a cross-chain ERC20 settlement via the `ChakraSettlementHandler::cross_chain_erc20_settlement()` function, the `to_handler` is not checked if it's whitelisted anywhere in the execution flow.

This leads to the possibility of users sending cross-chain messages to malicious handlers or handlers that were removed from the whitelist.

It also breaks a key invariant of the protocol stated in the ReadMe:
- The handler receiving a cross-chain message must be on the whitelist for the source chain.

## Proof of Concept
`to_handler` is not checked if it's whitelisted anywhere in the execution flow.

You can follow the execution flow:
`ChakraSettlementHandler::cross_chain_erc20_settlement()` -> `ChakraSettlement::send_cross_chain_msg()`

-> `ChakraSettlement::receive_cross_chain_msg()` -> `ChakraSettlementHandler::receive_cross_chain_msg()`

The only check is in `ChakraSettlementHandler::receive_cross_chain_msg`:
```
  if (is_valid_handler(from_chain, from_handler) == false) {
      return false;
  }
```
But this only checks if the source handler is whitelisted on the source chain. The `to_handler` i.e the handler receiving the cross-chain message is not checked anywhere.

And as stated in the invariant:
- The handler receiving a cross-chain message must be on the whitelist for the source chain

## Tools Used
Manual review
## Recommended Mitigation Steps
Add a check somewhere in the flow that the handler receiving a cross-chain message should be on the whitelist for the source chain.
