ToC
- Ethereum - StarkNet communication

- Aave-StarkNet-bridge
  - Ethereum to StarkNet
  - StarkNet to Ethereum


# Ethereum - StarkNet communication
## Ethereum to StarkNet (L1 - L2)
- a function of StarkNet core contract on Ethereum is called with l2 contract, function selector, payload
- StarkNet sequencer receipts the payload, initiates a transaction that invokes the correspond contract function. And the follow logics happend on StarkNet contract
- L1 state update and proof
  - The L1 Handler transaction that was created in the previous step is added to a proof.
  - The state update is received on the Core contract
  - the message is cleared from the Core contract’s storage. At this point the message is handled.


## StarkNet to Ethereum (L2 - L1)
- Interaction with contract on StarkNet
- On StarkNet contract, `send_message_to_l1` is called with `to_address` and `payload`
- When message is consumed
  - `consumeMessageFromL2` function is called in StarkNet core contract
  - proof

the hash of the message is stored on L1 in the StarkNet Core Contract (and the relevant counter is increased), and the LogMessageToL1 event (which contains the message parameters) is emitted. are these done by StarkNet OS/sequencer

# [aave-starknet-bridge](https://github.com/aave-starknet-project/aave-starknet-bridge)

The bridge allows users to deposit / withdraw aTokens (and only aTokens) on Ethereum side, then mints / burns the wrapped aTokens named static_a_tokens on StarkNet
- Deposit aTokens on Ethereum => mint static_a_tokens on Starknet
- Withdrawn aTokens to Ethereum => burn static_a_tokens on Starknet

## L1 - L2 deposit
### User deposits aTokens/underlying token to bridge [contract](https://github.com/aave-starknet-project/aave-starknet-bridge/blob/main/contracts/l1/Bridge.sol#L83) on Ethereum

### The bridge contract interacts with StarkNet core contract on Ethereum
- Send message to StarkNet core contract that includes:
  - `l2Bridge`: The recipient contract address on StarkNet
  - `Cairo.DEPOSIT_HANDLER`: Function selector
  - `payload`
  ```js
  _messagingContract.sendMessageToL2(
      _l2Bridge,
      Cairo.DEPOSIT_HANDLER,
      payload
  );
  ```
- `_messagingContract` - StarkNet core contract
  - On Mainnet: [0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4](https://etherscan.io/address/0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4#code)
  - Goerli: [0xde29d060D45901Fb19ED6C6e959EB22d8626708e](https://goerli.etherscan.io/address/0xde29d060D45901Fb19ED6C6e959EB22d8626708e#code)
  - [Code on Github](https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/eth/StarknetMessaging.sol)
  
- [sendMessageToL2 function](https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/eth/StarknetMessaging.sol#L100)
  - Message parameters
    - `toAddress`: the target bridge contract address on StarkNet
    - `selector`: Function selector <br />
    An uint256 which derived from function name `handle_deposit` of bridge contract on StarkNet <br />
    This function need to be annotated with the `l1_handler` decorator
    - `payload`: `uint256[](9)` that will be passed as arguments for function `handle_deposit`
  - hashes the message parameters
  - updates the L1→L2 message mapping `l1ToL2Messages()[msgHash] += 1;`

  ```js
  function sendMessageToL2(
      uint256 toAddress,
      uint256 selector,
      uint256[] calldata payload
  ) external override returns (bytes32) {
      uint256 nonce = l1ToL2MessageNonce();
      NamedStorage.setUintValue(L1L2_MESSAGE_NONCE_TAG, nonce + 1);
      emit LogMessageToL2(msg.sender, toAddress, selector, payload, nonce);
      bytes32 msgHash = getL1ToL2MsgHash(toAddress, selector, payload, nonce);
      l1ToL2Messages()[msgHash] += 1;
      return msgHash;
  }
  ```
  - `payload`
    ```js
    payload[0] = uint256(uint160(from)); //L1 deposit function caller address (user address)
    payload[1] = l2Recipient; // user define
    payload[2] = _aTokenData[l1Token].l2TokenAddress; // l2TokenAddress
    (payload[3], payload[4]) = Cairo.toSplitUint(amount); // split uint256 to 2 felts
    (payload[5], payload[6]) = Cairo.toSplitUint(blockNumber);
    (payload[7], payload[8]) = Cairo.toSplitUint(currentRewardsIndex);
    ```

### StarkNet sequencer
- Upon seeing enough L1 confirmations for the transaction that sent the message, initiates the corresponding L2 transaction.
- That L2 transaction invokes the `handle_deposit` function
- Arguments are from `payload` of L1 transaction

```rust
@l1_handler
func handle_deposit{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(
    from_address: felt,
    l1_sender: felt,
    l2_recipient: felt,
    l2_token: felt,
    amount_low: felt,
    amount_high: felt,
    block_number_low: felt,
    block_number_high: felt,
    l1_rewards_index_low: felt,
    l1_rewards_index_high: felt,
)
```

## L2 - L1 withdraw
To bridge their static_a_tokens back to L1, users initiate a withdrawal on the L2 bridge by calling `initiate_withdraw`

- The amount of static_a_tokens to withdraw will be burned by L2 bridge.
- A message will be sent to L1 bridge with L1 aToken address, L1 recipient, L2 rewards index and the amount.

```rust
@external
func initiate_withdraw{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(
    l2_token: felt, l1_recipient: felt, amount: Uint256, to_underlying_asset: felt
) {
  // the message
  let (message_payload: felt*) = alloc();
  assert message_payload[0] = WITHDRAW_MESSAGE; //const WITHDRAW_MESSAGE = 2;
  assert message_payload[1] = l1_token;
  assert message_payload[2] = caller_address;
  assert message_payload[3] = l1_recipient;
  assert message_payload[4] = amount.low;
  assert message_payload[5] = amount.high;
  assert message_payload[6] = current_rewards_index.wad.low;
  assert message_payload[7] = current_rewards_index.wad.high;
  assert message_payload[8] = to_underlying_asset;

  send_message_to_l1(to_address=to_address, payload_size=9, payload=message_payload);
  // to_address is L1 bridge contract
}
```
- [`send_message_to_l1` function](https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/common/messages.cairo)
- `_consumeMessage` of Bridge.sol will consume the message by calling StarkNet core contract [`consumeMessageFromL2`](https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/eth/StarknetMessaging.sol#L119)
  - this is an internal function, called by external function `withdraw`
  - so the message will only be consumed when `withdraw` is called
  - 

## L1 - L2 to update L2 state
`handle_index_update`, called when L1 bridge updateL2State function is called.

## L2 - L1 to receiveRewards


# References
- https://docs.starknet.io/documentation/develop/L1-L2_Communication/messaging-mechanism/