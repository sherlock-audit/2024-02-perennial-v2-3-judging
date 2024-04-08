Damp Coconut Jaguar

medium

# Vault checkpoints slightly incorrect conversion from assets to shares leads to slow loss of funds for long-time vault depositors

## Summary

When vault checkpoints convert assets to shares (specifically used to calculate user's shares for their deposit), it uses the following formula:
`shares = (assets[before fee] - settlementFee) * checkpoint.shares/checkpoint.assets * (deposit + redeem - tradeFee) / (deposit + redeem)`

`settlementFee` in this formula is taken into account slightly incorrectly: in actual market collateral calculations, both settlement fee and trade fee are subtracted from collateral, but this formula basically multiplies `1 - settlement fee percentage` by `1 - trade fee percentage`, which is slightly different and adds the calculation error = `settlement fee percentage * trade fee percentage`.

This is the scenario to better understand the issue:
1. Linear fee = 2%, settlement fee = $1
2. User1 deposits $100 into the vault (linear fee = $2, settlement fee = $1)
3. Vault assets = $97 (due to fees), User1 shares = 100
4. User2 deposits $100 into the vault (linear fee = $2, settlement fee = $1)
5. Vault assets = $194, User1 shares = 100, but User2 shares = 100.02, meaning User1's share value has slightly fallen due to a later deposit.

This is the calculation for User2 shares:
`shares = ($100 - $1) * 100/$97 * ($100 - $2) / $100 = $99 * 100/$97 * $98/$100 = $99 * 98/$97 = 100.02`

The extra 0.02 this user has received is because the `tradeFee` is taken from the amount after settlement fee ($99) rather than full amount as it should ($100). This difference (`settlementFee * tradeFee = $0.02`) is unfair amount earned by User2 and loss of funds for User1.

When redeeming, the formula for shares -> assets vault checkpoint conversion is correct and the correct amount is redeemed. 

This issue leads to all vault depositors slowly losing share value with each deposit, and since no value is gained when redeeming, continuous deposits and redeems will lead to all long-time depositors continuously losing their funds.

## Vulnerability Detail

This is the formula for vault checkpoint `toSharesGlobal`:
```solidity
function toSharesGlobal(Checkpoint memory self, UFixed6 assets) internal pure returns (UFixed6) {
    // vault is fresh, use par value
    if (self.shares.isZero()) return assets;

    // if vault is insolvent, default to par value
    return  self.assets.lte(Fixed6Lib.ZERO) ? assets : _toShares(self, _withoutSettlementFeeGlobal(self, assets));
}

function _toShares(Checkpoint memory self, UFixed6 assets) private pure returns (UFixed6) {
    UFixed6 selfAssets = UFixed6Lib.unsafeFrom(self.assets);
    return _withSpread(self, assets.muldiv(self.shares, selfAssets));
}

function _withSpread(Checkpoint memory self, UFixed6 amount) private pure returns (UFixed6) {
    UFixed6 selfAssets = UFixed6Lib.unsafeFrom(self.assets);
    UFixed6 totalAmount = self.deposit.add(self.redemption.muldiv(selfAssets, self.shares));
    UFixed6 totalAmountIncludingFee = UFixed6Lib.unsafeFrom(Fixed6Lib.from(totalAmount).sub(self.tradeFee));

    return totalAmount.isZero() ?
        amount :
        amount.muldiv(totalAmountIncludingFee, totalAmount);
}

function _withoutSettlementFeeGlobal(Checkpoint memory self, UFixed6 amount) private pure returns (UFixed6) {
    return _withoutSettlementFee(amount, self.settlementFee);
}

function _withoutSettlementFee(UFixed6 amount, UFixed6 settlementFee) private pure returns (UFixed6) {
    return amount.unsafeSub(settlementFee);
}
```

This code translates to a formula shown above, i.e. it first subtracts settlement fee from the assets (`withoutSettlementFeeGlobal`), then multiplies this by checkpoint's share value in `_toShares` (`*checkpoint.shares/checkpoint.assets`), and then multiplies this by trade fee adjustment in `_withSpread` (`*(deposit+redeem-tradeFee) / (deposit+redeem)`). Here is the formula again:
`shares = (assets[before fee] - settlementFee) * checkpoint.shares/checkpoint.assets * (deposit + redeem - tradeFee) / (deposit + redeem)`

As shown above, the formula is incorrect, because it basically does the following:
`user_assets = (deposit - settlementFee) * (deposit - tradeFee)/deposit = deposit * (1 - settlementFeePct) * (1 - tradeFeePct)`

But the actual user collateral after fees is calculated as:
`user_assets = deposit - settlementFee - tradeFee = deposit * (1 - settlementFeePct - tradeFeePct)`

If we subtract the actual collateral from the formula used in checkpoint, we get the error:
`error = deposit * ((1 - settlementFeePct) * (1 - tradeFeePct) - (1 - settlementFeePct - tradeFeePct))`
`error = deposit * settlementFeePct * tradeFeePct`
`error = settlementFee * tradeFeePct`

So this is systematic error, which inflates the shares given to users with any deposit by fixed amount of `settlementFee * tradeFeePct`

## Impact

Any vault deposit reduces the vault assets by `settlementFee * tradeFeePct`. While this amount is not very large (in the order of $0.1 - $0.001 per deposit transaction), this is amount lost with each deposit, and given that an active vault can easily have 1000s of transactions daily, this will be a loss of $1-$100/day, which is significant enough to make it a valid issue.

## Code Snippet

SettlementFee subtracted from asset before proceeding in `toSharesGlobal`:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L91-L97

The result is multiplied by the checkpoint's share to assets ratio in `_toShares`:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L153-L156

And the final result is multiplied by `tradeFee`-adjusted deposits and redeems in `_withSpread`:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Checkpoint.sol#L169-L177

## Tool used

Manual Review

## Recommendation

Re-work the assets to shares conversion in vault checkpoint to use the correct formula:
`shares = (assets[before fee] - settlementFee - tradeFee * assets / (deposit + redeem)) * checkpoint.shares/checkpoint.assets`