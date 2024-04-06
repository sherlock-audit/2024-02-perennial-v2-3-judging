Swift Cornflower Manatee

medium

# Liquidator can set up referrals for other users

## Summary
If a user has met the liquidation criteria and currently has no referrer
then a malicious liquidator can specify a referrer in the liquidation order.
making it impossible for subsequent users to set up the referrer they want.

## Vulnerability Detail
Currently, there are 2 conditions to set up a referrer
1. the order cannot be empty   （Non-empty orders require authorization unless they are liquidation orders）
2. there can't be another referrer already  

```solidity
    function _loadUpdateContext(
        Context memory context,
        address account,
        address referrer
    ) private view returns (UpdateContext memory updateContext) {
...
        updateContext.referrer = referrers[account][context.local.currentId];
        updateContext.referralFee = IMarketFactory(address(factory())).referralFee(referrer);
    }

    function _processReferrer(
        UpdateContext memory updateContext,
        Order memory newOrder,
        address referrer
    ) private pure {
@>      if (newOrder.makerReferral.isZero() && newOrder.takerReferral.isZero()) return;
        if (updateContext.referrer == address(0)) updateContext.referrer = referrer;
        if (updateContext.referrer == referrer) return;

        revert MarketInvalidReferrerError();
    }


    function _storeUpdateContext(Context memory context, UpdateContext memory updateContext, address account) private {
...
        referrers[account][context.local.currentId] = updateContext.referrer;
    }
```
However, if the user does not have a referrer, the liquidation order is able to meet both of these restrictions

This allows the liquidator to set up referrals for other users.

When the user subsequently tries to set up a referrer, it will fail.

## Impact

If a user is set up as a referrer by a liquidated order in advance, the user cannot be set up as anyone else.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L503
## Tool used

Manual Review

## Recommendation
Restrictions on Liquidation Orders Cannot Set a referrer
```diff
    function _processReferrer(
        UpdateContext memory updateContext,
        Order memory newOrder,
        address referrer
    ) private pure {
+       if (newOrder.protected() && referrer != address(0)) revert MarketInvalidReferrerError;
        if (newOrder.makerReferral.isZero() && newOrder.takerReferral.isZero()) return;
        if (updateContext.referrer == address(0)) updateContext.referrer = referrer;
        if (updateContext.referrer == referrer) return;

        revert MarketInvalidReferrerError();
    }
```