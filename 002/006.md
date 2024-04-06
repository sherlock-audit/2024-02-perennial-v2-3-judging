Damp Coconut Jaguar

medium

# Requested oracle versions, which have expired, must return this oracle version as invalid, but they return it as a normal version with previous version's price instead

## Summary

Each market action requests a new oracle version which must be commited by the keepers. However, if keepers are unable to commit requested version's price (for example, no price is available at the time interval, network or keepers are down), then after a certain timeout this oracle version will be commited as invalid, using the previous valid version's price.

The issue is that when this expired oracle version is used by the market (using `oracle.at`), the version returned will be valid (`valid = true`), because oracle returns version as invalid only if `price = 0`, but the `commit` function sets the previous version's price for these, thus it's not 0.

This leads to market using invalid versions as if they're valid, keeping the orders (instead of invalidating them), which is a broken core functionality and a security risk for the protocol.

## Vulnerability Detail

When requested oracle version is commited, but is expired (commited after a certain timeout), the price of the previous valid version is set to this expired oracle version:
```solidity
function _commitRequested(OracleVersion memory version) private returns (bool) {
    if (block.timestamp <= (next() + timeout)) {
        if (!version.valid) revert KeeperOracleInvalidPriceError();
        _prices[version.timestamp] = version.price;
    } else {
        // @audit previous valid version's price is set for expired version
        _prices[version.timestamp] = _prices[_global.latestVersion]; 
    }
    _global.latestIndex++;
    return true;
}
```

Later, `Market._processOrderGlobal` reads the oracle version using the `oracle.at`, invalidating the order if the version is invalid:
```solidity
function _processOrderGlobal(
    Context memory context,
    SettlementContext memory settlementContext,
    uint256 newOrderId,
    Order memory newOrder
) private {
    OracleVersion memory oracleVersion = oracle.at(newOrder.timestamp);

    context.pending.global.sub(newOrder);
    if (!oracleVersion.valid) newOrder.invalidate();
```

However, expired oracle version will return `valid = true`, because this flag is only set to `false` if `price = 0`:
```solidity
function at(uint256 timestamp) public view returns (OracleVersion memory oracleVersion) {
    (oracleVersion.timestamp, oracleVersion.price) = (timestamp, _prices[timestamp]);
    oracleVersion.valid = !oracleVersion.price.isZero(); // @audit <<< valid = false only if price = 0
}
```

This means that `_processOrderGlobal` will treat this expired oracle version as valid and won't invalidate the order.

## Impact

Market uses invalid (expired) oracle versions as if they're valid, keeping the orders (instead of invalidating them), which is a broken core functionality and a security risk for the protocol.

## Code Snippet

`KeeperOracle._commitRequested` sets `_prices` to the last valid version's price for expired versions:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L153-L162

`Market._processOrderGlobal` reads the oracle version using the `oracle.at`, invalidating the order if the version is invalid:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L604-L613

`KeeperOracle.at` returns `valid = false` only if `price = 0`, but since expired version has valid price, it will be returned as a valid version:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L109-L112

## Tool used

Manual Review

## Recommendation

Add validity map along with the price map to `KeeperOracle` when recording commited price.