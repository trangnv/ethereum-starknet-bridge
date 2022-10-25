# Ethereum <> Starknet

## [aave-starknet-bridge](https://github.com/aave-starknet-project/aave-starknet-bridge)

The bridge allows users to deposit / withdraw their aTokens (and only aTokens) on Ethereum side, then mints / burns them wrapped aTokens named static_a_tokens on StarkNet
- Deposit aTokens on Ethereum => mint static_a_tokens on Starknet
- Withdrawn aTokens to Ethereum => burn static_a_tokens on Starknet

Some noteable properties of `static_a_tokens`
- Increasing value over time by token value growing, when aTokens grow in balance.
- Allows claiming an accruing amount of Aave reward tokens.

### Deposit aTokens/underlying token on Ethereum
Checks
- Check total deposit doesn't reach ceiling
- Check underlying asset has been approved
- Check deposit amount > 0
- User deposit underlying asset or aTokens
  - If fromUnderlyingAsset: transfer underlying asset
  - Else: transfer l1AToken

#### Interact with StarkNet core contract on Ethereum
- Send message with `_sendDepositMessage`
```js
_messagingContract.sendMessageToL2(
    _l2Bridge,
    Cairo.DEPOSIT_HANDLER,
    payload
);
```
Where:
  - `l2Bridge`: The recipient contract address on StarkNet
  - `Cairo.DEPOSIT_HANDLER`: Function selector
  - `payload`:

- Messaging contract
  - The StarkNet core contract on Mainnet: [0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4](https://etherscan.io/address/0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4#code)
  - The StarkNet core contract on Goerli: [0xde29d060D45901Fb19ED6C6e959EB22d8626708e](https://goerli.etherscan.io/address/0xde29d060D45901Fb19ED6C6e959EB22d8626708e#code)
  - [Code on Github](https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/eth/StarknetMessaging.sol)
  
- [sendMessageToL2 function](https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/eth/StarknetMessaging.sol#L100)

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
  - hashes the message parameters
  - updates the L1→L2 message mapping `l1ToL2Messages()[msgHash] += 1;`


#### StarkNet sequencer handle the message
- consumes the message and invokes the requested L2 function of the contract designated by the “to” address.
- The sequenser is close source?
- 


#### Proof added
The L1 Handler transaction that was created in the previous step is added to a proof.

#### Core contract on Ethereum get state update

#### Clear message
the message is cleared from the Core contract’s storage. At this point the message is handled.



## References
- https://docs.starknet.io/documentation/develop/L1-L2_Communication/messaging-mechanism/