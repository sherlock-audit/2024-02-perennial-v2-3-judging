Swift Cornflower Manatee

medium

# _loadContext() uses the wrong pendingGlobal.

## Summary
`StrategyLib._loadContext()` is using the incorrect `pendingGlobal`, causing `currentPosition`, `minPosition`, and `maxPosition` to be incorrect, leading to incorrect rebalance operation.

## Vulnerability Detail
In `StrategyLib._loadContext()`, there is a need to compute `currentPosition`, `minPosition`, and `maxPosition`. 
The code  as follows:
```solidity
    function _loadContext(
        Registration memory registration
    ) private view returns (MarketStrategyContext memory marketContext) {
...
        // current position
@>      Order memory pendingGlobal = registration.market.pendings(address(this));
        marketContext.currentPosition = registration.market.position();
        marketContext.currentPosition.update(pendingGlobal);
        marketContext.minPosition = marketContext.currentAccountPosition.maker
            .unsafeSub(marketContext.currentPosition.maker
                .unsafeSub(marketContext.currentPosition.skew().abs()).min(marketContext.closable));
        marketContext.maxPosition = marketContext.currentAccountPosition.maker
            .add(marketContext.riskParameter.makerLimit.unsafeSub(marketContext.currentPosition.maker));
    }
```

The code above `pendingGlobal = registration.market.pendings(address(this));` is wrong
It takes the address(this)'s `pendingLocal`.
The correct approach is to use `pendingGlobal = registration.market.pending();`.

## Impact
Since `pendingGlobal` is wrong, `currentPosition`, `minPosition` and `maxPosition` are all wrong.
affects subsequent rebalance calculations, such as `target.position` etc.
rebalance does not work properly

## Code Snippet
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L200
## Tool used

Manual Review

## Recommendation
```diff
    function _loadContext(
        Registration memory registration
    ) private view returns (MarketStrategyContext memory marketContext) {
...
        // current position
-       Order memory pendingGlobal = registration.market.pendings(address(this));
+       Order memory pendingGlobal = registration.market.pending();
        marketContext.currentPosition = registration.market.position();
        marketContext.currentPosition.update(pendingGlobal);
        marketContext.minPosition = marketContext.currentAccountPosition.maker
            .unsafeSub(marketContext.currentPosition.maker
                .unsafeSub(marketContext.currentPosition.skew().abs()).min(marketContext.closable));
        marketContext.maxPosition = marketContext.currentAccountPosition.maker
            .add(marketContext.riskParameter.makerLimit.unsafeSub(marketContext.currentPosition.maker));
    }
```
