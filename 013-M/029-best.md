Swift Cornflower Manatee

medium

# ChainlinkFactory no convert  decimals

## Summary
`ChainlinkFactory._parsePrices ()` parsing `verifiedReports` returns 8 decimals and is not converted to 18 decimals
## Vulnerability Details
`ChainlinkFactory._parsePrices ()` parses the `verifiedReports`, the code as follows:
```solidity
    function _parsePrices(
        bytes32[] memory ids,
        bytes calldata data
    ) internal override returns (PriceRecord[] memory prices) {
        bytes[] memory verifiedReports = chainlink.verifyBulk{value: msg.value}(
            abi.decode(data, (bytes[])),
            abi.encode(feeTokenAddress)
        );
        if (verifiedReports.length != ids.length) revert ChainlinkFactoryInputLengthMismatchError();

        prices = new PriceRecord[](ids.length);
        for (uint256 i = 0; i < verifiedReports.length; i++) {
            (bytes32 feedId, , uint32 observationsTimestamp, , , , uint192 price) =
@>              abi.decode(verifiedReports[i], (bytes32, uint32, uint32, uint192, uint192, uint32, uint192));

            if (feedId != toUnderlyingId[ids[i]]) revert ChainlinkFactoryInvalidFeedIdError(feedId);

@>          prices[i] = PriceRecord(observationsTimestamp, Fixed18Lib.from(UFixed18.wrap(price)));
        }
    }
```
However, according to the chainlink documentation, the format of verifiedReports is as follows:

https://docs.chain.link/data-streams/reference/interfaces
```solidity
contract StreamsUpkeep is ILogAutomation, StreamsLookupCompatibleInterface {
    struct BasicReport {
        bytes32 feedId; // The feed ID the report has data for
        uint32 validFromTimestamp; // Earliest timestamp for which price is applicable
        uint32 observationsTimestamp; // Latest timestamp for which price is applicable
        uint192 nativeFee; // Base cost to validate a transaction using the report, denominated in the chain’s native token (WETH/ETH)
        uint192 linkFee; // Base cost to validate a transaction using the report, denominated in LINK
        uint32 expiresAt; // Latest timestamp where the report can be verified onchain
 @>     int192 price; // DON consensus median price, carried to 8 decimal places
    }
...
```

`price` is 8 decimal, not 18 decimal `UFixed18`

and the `price` is `int192` not  `uint192`. In extreme cases, there may be a price of `< 0`

## Impact

The wrong price unit may cause abnormal market calculations or an error in the `MultiInvoker`'s order price trigger condition

## Code Snippet
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/chainlink/ChainlinkFactory.sol#L60
## Tool used

Manual Review

## Recommendation
1. Convert to 18 decimal
2. check that the price cannot be less than 0
```diff
    function _parsePrices(
        bytes32[] memory ids,
        bytes calldata data
    ) internal override returns (PriceRecord[] memory prices) {
        bytes[] memory verifiedReports = chainlink.verifyBulk{value: msg.value}(
            abi.decode(data, (bytes[])),
            abi.encode(feeTokenAddress)
        );
        if (verifiedReports.length != ids.length) revert ChainlinkFactoryInputLengthMismatchError();

        prices = new PriceRecord[](ids.length);
        for (uint256 i = 0; i < verifiedReports.length; i++) {
-           (bytes32 feedId, , uint32 observationsTimestamp, , , , uint192 price) =
-               abi.decode(verifiedReports[i], (bytes32, uint32, uint32, uint192, uint192, uint32, uint192));
+           (bytes32 feedId, , uint32 observationsTimestamp, , , ,  int192 price) =
+               abi.decode(verifiedReports[i], (bytes32, uint32, uint32, uint192, uint192, uint32, int192));
+           price = price * 10**10; // 8 decimal to 18 decimal
+           require(price>=0,"invalid price");
            if (feedId != toUnderlyingId[ids[i]]) revert ChainlinkFactoryInvalidFeedIdError(feedId);

            prices[i] = PriceRecord(observationsTimestamp, Fixed18Lib.from(UFixed18.wrap(price)));
        }
    }
```