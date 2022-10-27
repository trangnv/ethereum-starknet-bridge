ToC

1. Aave-StarkNet-bridge
1.1. Ethereum to StarkNet
1.2. StarkNet to Ethereum

2. More on Ethereum - StarkNet communication

3. Aave portal feature

# 1. [aave-starknet-bridge](https://github.com/aave-starknet-project/aave-starknet-bridge)

The bridge allows users to deposit / withdraw aTokens (and only aTokens) on Ethereum side, then mints / burns the wrapped aTokens named static_a_tokens on StarkNet
- Deposit aTokens on Ethereum => mint static_a_tokens on Starknet
- Withdrawn aTokens to Ethereum => burn static_a_tokens on Starknet

Some noteable properties of `static_a_tokens`
- Increasing value over time by token value growing, when aTokens grow in balance.
- Allows claiming an accruing amount of Aave reward tokens.

## 1.1. Ethereum to StarkNet (L1 - L2)
### 1.1.1. Deposit aTokens/underlying token on [Ethereum contract](https://github.com/aave-starknet-project/aave-starknet-bridge/blob/main/contracts/l1/Bridge.sol#L83)
Checks
- Check total deposit doesn't reach ceiling
- Check underlying asset has been approved
- Check deposit amount > 0
- User deposit underlying asset or aTokens
  - If fromUnderlyingAsset: transfer underlying asset
  - Else: transfer l1AToken

#### Interact with StarkNet core contract on Ethereum
- Send message with `_sendDepositMessage` that includes:
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
- StarkNet core contract
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

#### StarkNet sequencer
- upon seeing enough L1 confirmations for the transaction that sent the message, initiates the corresponding L2 transaction.
- The L2 transaction invokes the `handle_deposit` function

#### Proof added
The L1 Handler transaction that was created in the previous step is added to a proof.

#### Core contract on Ethereum get state update

#### Clear message
the message is cleared from the Core contract’s storage. At this point the message is handled.

### 1.1.2. Get static_a_tokens minted on StarkNet
`handle_deposit`
Function that mints L2 static aTokens and updates its state (latest L1 update block number and L1 rewards index).
- Initiated and revoked by the sequenser
- Arguments are from `payload` from `sendMessageToL2`
- Check:
  - `only_l1_bridge(from_address_=from_address);`
  - construct `amount`, `l1_rewards_index`, `block_number` to `Uint256` type
  - assert_not_zero(l2_token);
- handle the difference of the index at send and receive
- Call mint on l2_token contract.
- 


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


### 1.2.3. Reward on L2
- Starknet users will keep enjoying the same rewards as on L1 (AAVE token) after bridging
- To do so, L1 rewards index is stored in the state of `static_a_tokens`.
- The index is updated 
  - Every time a user deposits or withdraw the corresponding aToken, 
  - Can also be updated in a permissionless manner by calling the function `updateL2State` in L1 bridge.
- Claiming rewards
  - when withdrawing static_a_tokens from L2 
  - call `claim_rewards` on `static_a_token` contract => calls L2 bridge => L2 bridge mints due rewAAVE tokens to the L2 user.

### `static_a_token`
## 1.2. StarkNet to Ethereum (L2 - L1)

# 2. More on Ethereum - StarkNet communication

# 3. Aave portal feature

## References
- https://docs.starknet.io/documentation/develop/L1-L2_Communication/messaging-mechanism/