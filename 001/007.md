Tame Pebble Shark

medium

# Use of `call()` Instead of `transfer()` in `MultiInvoker` 's `invoke` Function

## Summary
The `MultiInvoker` contract employs the `transfer()` function when transferring Ether to recipients within its `invoke()` function. However, this approach can be problematic when the recipient is a smart contract with specific requirements for ETH transfers. To address this, the contract should utilize `call()` instead of `transfer()` to ensure compatibility with a wider range of smart contracts.


## Vulnerability Detail
The `invoke()` function in the `MultiInvoker` contract aims to execute a series of actions defined in Invocation structs. These actions can involve interacting with other contracts, including transferring Ether. Currently, the function employs `transfer()` to send Ether back to the caller after executing the invocations. However, `transfer()` has limitations when transferring ETH to smart contracts:
- If the recipient contract does not have a payable fallback function, the transfer will fail.
- If the payable fallback function consumes more than 2300 gas units, the transfer will fail.
- If the payable fallback function consumes less than 2300 gas units but is called through a proxy that raises the call’s gas usage above 2300, the transfer will fail.
Here's a relevant code snippet from the `invoke()` function:

```solidity
// Eth must not remain in this contract at rest
payable(msg.sender).transfer(address(this).balance);
```
## Impact
The use of `transfer()` in the `invoke()` function can lead to failed ETH transfers, disrupting the intended functionality of the contract. This could result in funds being locked within the contract or transactions failing unexpectedly.


## Code Snippet
[MultiInvoker.sol#L163](https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L163)
## Tool used

Manual Review

## Recommendation
The `MultiInvoker` contract should replace `transfer()` with `call()` when transferring Ether back to the caller. 
```solidity
// Eth must not remain in this contract at rest
(bool success, ) = msg.sender.call{value: address(this).balance}("");
require(success, "Transfer failed");
```