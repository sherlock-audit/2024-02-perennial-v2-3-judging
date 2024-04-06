Damp Coconut Jaguar

medium

# Vault and oracle keepers DoS in some situations due to `market.update(account,max,max,max,0,false)`

## Summary

When user's market account is updated without position and collateral change (by calling `market.update(account,max,max,max,0,false)`), this serves as some kind of "settling" the account (which was the only way to settle the account before v2.3). However, this action still reverts if the account is below margin requirement.

The issue is that some parts of the code use this action to "settle" the account in the assumption that it never reverts which is not true. This causes unpexpected reverts and denial of service to users who can not execute transactions in some situations, in particular:

1. Oracle `KeeperFactory.settle` uses this method to settle all accounts in the market for the oracle verison and will revert entire market version's settlement if any account which is being settled is below margin requirement. Example scenario:
1.1. User increases position to the edge of margin requirement
1.2. The price rises slightly for the commited oracle version, and user position is settled and is now slightly below margin requirements
1.3. All attempts to settle accounts for the commited oracle version for this market will revert as user's account collateral is below margin requirements.

2. Vault `Vault._updateUnderlying` uses this method to settle all vault's accounts in the markets. This function is called at the start of `rebalance` and `update`, with `rebalance` also being called before any admin vault parameters changes such as updating market leverages, weights or cap. This becomes especially problematic if any market is "removed" from the vault by setting its weight to 0, but the market still has some position due to `minPosition` limitation (as described in another issue). In such case each vault update will bring this market's position to exact edge of margin requirement, meaning a lot of times minimal price changes will put the vault's market account below margin requirement, and as such most Vault functions will revert (`update`, `rebalance` and admin param changes). Moreover, since the vault rebalances collateral and/or position size only in `_manage` (which is called only from `update` and `rebalance`), this means that the vault is basically bricked until this position is either liquidated or goes above margin requirement again due to price changes.

## Vulnerability Detail

When `Market.update` is called, any parameters except `protected = true` will perform the following check from the `InvariantLib.validate`:
```solidity
if (
    !PositionLib.margined(
        context.latestPosition.local.magnitude().add(context.pending.local.pos()),
        context.latestOracleVersion,
        context.riskParameter,
        context.local.collateral
    )
) revert IMarket.MarketInsufficientMarginError();
```

This means that even updates which do not change anything (empty order and 0 collateral change) still perform this check and revert if the user's collateral is below margin requirement.

Such method to settle accounts is used in `KeeperOracle._settle`:
```solidity
function _settle(IMarket market, address account) private {
    market.update(account, UFixed6Lib.MAX, UFixed6Lib.MAX, UFixed6Lib.MAX, Fixed6Lib.ZERO, false);
}
```

This is called from `KeeperFactory.settle`, which the keepers are supposed to call to settle market accounts after the oracle version is commited. This will revert, thus keepers will temporarily be unable to call this function for the specific oracle version until all users are at or above margin.

The same method is used to settle accounts in `Vault._updateUnderlying`:
```solidity
function _updateUnderlying() private {
    for (uint256 marketId; marketId < totalMarkets; marketId++)
        _registrations[marketId].read().market.update(
            address(this),
            UFixed6Lib.MAX,
            UFixed6Lib.ZERO,
            UFixed6Lib.ZERO,
            Fixed6Lib.ZERO,
            false
        );
}
```

## Impact

1. Keepers are unable to settle market accounts for the commited oracle version until all accounts are above margin. The oracle fees are still taken from all accounts, but the keepers are blocked from receiving it.
2. If any Vault's market weight is set to 0 (or if vault's position in any market goes below margin for whatever other reason), most of the time the vault will temporarily be bricked until vault's position in that market is liquidated. The only function working in this state is `Vault.settle`, even all admin functions will revert.

## Code Snippet

`InvariantLib.validate` reverts for all updates (except liquidations) where account is below margin requirements:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial/contracts/libs/InvariantLib.sol#L78-L85

`KeeperOracle._settle` uses `Market.update` to settle accounts:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L178-L180

`Vault._updateUnderlying` also uses the same method to settle accounts:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L342-L352

## Tool used

Manual Review

## Recommendation

Depending on intended functionality:
1. Ignore the margin requirement for empty orders and collateral change which is >= 0.
**AND/OR**
2. Use `Market.settle` instead of `Market.update` to settle accounts, specifically in `KeeperOracle._settle` and in `Vault._updateUnderlying`. There doesn't seem to be any reason or issue to use `settle` instead of `update`, it seems that `update` is there just because there was no `settle` function available before.