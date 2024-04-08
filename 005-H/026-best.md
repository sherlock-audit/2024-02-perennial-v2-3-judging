Damp Coconut Jaguar

high

# Vault global shares and assets change will mismatch local shares and assets change during settlement due to incorrect `_withoutSettlementFeeGlobal` formula

## Summary

Every vault update, which involves change of position in the underlying markets, `settlementFee` is charged by the Market. Since many users can deposit and redeem during the same oracle version, this `settlementFee` is shared equally between all users of the same oracle version. However, there is an issue in that `settlementFee` is charged once both for deposits and redeems, however `_withoutSettlementFeeGlobal` subtracts `settlementFee` in full both for deposits and redeems, meaning that for global fee, it's basically subtracted twice (once for deposits, and another time for redeems). But for local fee, it's subtracted proportional to `checkpoint.orders`, with sum of fee subtracted equal to exactly `settlementFee` (once). This difference in global and local `settlementFee` calculations leads to inflated `shares` and `assets` added for user deposits (local state) compared to vault overall (global state).

## Vulnerability Detail

Here is an easy scenario to demonstrate the issue:
1. `SettlementFee = $10`
2. User1 deposits `$10` for oracle version `t = 100`
3. User2 redeems `10 shares` (worth `$10`) for the same oracle version `t = 100` (`checkpoint.orders = 2`)
4. Once the oracle version `t = 100` settles, we have the following:
4.1. Global deposits = $10, redeems = $10
4.2. Global deposits convert to `0 shares` (because `_withoutSettlementFeeGlobal(10)` applies `settlementFee` of $10 in full, returning `10-10=0`)
4.3. Global redeems convert to `0 assets` (because `_withoutSettlementFeeGlobal(10)` applies `settlementFee` of $10 in full, returning `10-10=0`)
4.4. User1 deposit of $10 converts to `5 shares` (because `_withoutSettlementFeeLocal(10)` applies `settlementFee` of $5 (because there are 2 orders), returning `10-5=5`)
4.5. User2 redeem of 10 shares converts to `$5` (for the same reason)

From the example above it can be seen that:
1. User1 receives 5 shares, but global vault shares didn't increase. Over time this difference will keep growing potentially leading to a situation where many user redeems will lead to 0 global shares, but many users will still have local shares which they will be unable to redeem due to underflow, thus losing funds.
2. User2's assets which he can claim increase by $5, but global claimable assets didn't change, meaning User2 will be unable to claim assets due to underflow when trying to decrease global assets, leading to loss of funds for User2.

The underflow in both cases will happen in `Vault._update` when trying to update global account:
```solidity
function update(
    Account memory self,
    uint256 currentId,
    UFixed6 assets,
    UFixed6 shares,
    UFixed6 deposit,
    UFixed6 redemption
) internal pure {
    self.current = currentId;
    // @audit global account will have less assets and shares than sum of local accounts
    (self.assets, self.shares) = (self.assets.sub(assets), self.shares.sub(shares));
    (self.deposit, self.redemption) = (self.deposit.add(deposit), self.redemption.add(redemption));
}
```

## Impact

Any time there are both deposits and redeems in the same oracle version, the users receive more (local) shares and assets than overall vault shares and assets increase (global). This mismatch causes:
1. Systematic increase of (sum of user shares - global shares), which can lead to bank run since the last users who try to redeem will be unable to do so due to underflow.
2. Systematic increase of (sum of user assets - global assets), which will lead to users being unable to claim their redeemed assets due to underflow.

The total difference in local and global `shares+assets` equals to `settlementFee` per each oracle version with both deposits and redeems. This can add up to significant amounts (at `settlementFee = $1` this can be $100-$1000 per day), meaning it will quickly become visible especially for point 2., because typically global claimable assets are at or near 0 most of the time, since users usually redeem and then immediately claim, thus any difference of global and local assets will quickly lead to users being unable to claim.

## Code Snippet

SettlementFee subtracted in `_withoutSettlementFeeGlobal`
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L183-L185

This is subtracted twice: for deposit and for redeem:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Account.sol#L62-L63

## Tool used

Manual Review

## Recommendation

Calculate total orders to deposit and total orders to redeem (in addition to total orders overall). Then `settlementFee` should be multiplied by `deposit/orders` for `toGlobalShares` and by `redeems/orders` for `toGlobalAssets`. This weightening of `settlementFee` will make it in-line with local order weights.