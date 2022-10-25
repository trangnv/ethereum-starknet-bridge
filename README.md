# Ethereum <> Starknet Aave portal

## aave-starknet-bridge

The bridge allows users to deposit / withdraw their aTokens (and only aTokens) on Ethereum side, then mints / burns them wrapped aTokens named static_a_tokens on StarkNet
- Deposit aTokens on Ethereum => mint static_a_tokens on Starknet
- Withdrawn aTokens to Ethereum => burn static_a_tokens on Starknet

Some noteable properties of `static_a_tokens`
- Increasing value over time by token value growing, when aTokens grow in balance.
- Allows claiming an accruing amount of Aave reward tokens.

### Contracts
#### L1 contract methods

##### Deposit
- Checks
  - Check total deposit doesn't reach ceiling
  - Check underlying asset has been approved
  - Check deposit amount > 0
  - User deposit underlying asset or aTokens
    - If fromUnderlyingAsset: transfer underlying asset
    - Else: transfer l1AToken

- Send message to L2 with `_sendDepositMessage`
```js
_messagingContract.sendMessageToL2(
    _l2Bridge,
    Cairo.DEPOSIT_HANDLER,
    payload
);
```
    - `l2Bridge`: The recipient contract address on StarkNet
    - `Cairo.DEPOSIT_HANDLER`: Function selector
    - `payload`:

- Messaging contract
  - The StarkNet core contract on Mainnet: [0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4](https://etherscan.io/address/0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4#code)
  - 





## 