Damp Coconut Jaguar

medium

# All transactions to claim assets from the vault will revert in some situations due to double subtraction of the claimed assets in market position allocations calculation.

## Summary

When assets are claimed from the vault (`Vault.update(0,0,x)` called), the vault rebalances its collateral. There is an issue with market positions allocation calculations: the assets ("total position") subtract claimed amount twice. This leads to revert in case this incorrect `assets` amount is less than `minAssets` (caused by market's `minPosition`). In situations when the vault can't redeem due to some market's position being at the `minPosition` (because of the market's skew, which disallows makers to reduce their positions), this will lead to all users being unable to claim any assets which were already redeemed and settled.

## Vulnerability Detail

`Vault.update` rebalances collateral by calling `_manage`:
```solidity
_manage(context, depositAssets, claimAmount, !depositAssets.isZero() || !redeemShares.isZero());
```

In the rebalance calculations, collateral and assets (assets here stands for "total vault position") are calculated as following:
```solidity
  UFixed6 collateral = UFixed6Lib.unsafeFrom(strategy.totalCollateral).add(deposit).unsafeSub(withdrawal);
  UFixed6 assets = collateral.unsafeSub(ineligable);

  if (collateral.lt(strategy.totalMargin)) revert StrategyLibInsufficientCollateralError();
  if (assets.lt(strategy.minAssets)) revert StrategyLibInsufficientAssetsError();
```

`ineligable` is calculated as following:
```solidity
function _ineligable(Context memory context, UFixed6 withdrawal) private pure returns (UFixed6) {
    // assets eligable for redemption
    UFixed6 redemptionEligable = UFixed6Lib.unsafeFrom(context.totalCollateral)
        .unsafeSub(withdrawal)
        .unsafeSub(context.global.assets)
        .unsafeSub(context.global.deposit);

    return redemptionEligable
        // approximate assets up for redemption
        .mul(context.global.redemption.unsafeDiv(context.global.shares.add(context.global.redemption)))
        // assets pending claim
        .add(context.global.assets)
        // assets withdrawing
        .add(withdrawal);
}
```

Notice that `ineligable` adds `withdrawal` in the end (which is the assets claimed by the user). Now back to collateral and assets calculation:
- `collateral = totalCollateral + deposit - withdrawal`
- `assets = collateral - ineligable = collateral - (redemptionEligable * redemption / (redemption + shares) + global.assets + withdrawal)`
- `assets = totalCollateral + deposit - withdrawal - [redemptionIneligable] - global.assets - withdrawal`
- `assets = totalCollateral + deposit - [redemptionIneligable] - global.assets - 2 * withdrawal`

See that `withdrawal` (assets claimed by the user) is subtracted twice in assets calculations. This means that assets calculated are smaller than it should. In particular, assets might become less than minAssets thus reverting in the following line:
```solidity
  if (assets.lt(strategy.minAssets)) revert StrategyLibInsufficientAssetsError();
```

Possible scenario for this issue to cause inability to claim funds:
1. Some vault market's has a high skew (|long - short|), which means that minimum maker position is limited by the skew.
2. User redeems large amount from the vault, reducing vault's position in that market so that market maker ~= |long - short|. This means that further redeems from the vault are not possible because the vault can't reduce its position in the market.
3. After that, the user tries to claim what he has redeemed, but all attempts to redeem will revert (both for this user and for any other user that might want to claim)

## Impact

In certain situations (redeem not possible from the vault due to high skew in some underlying market) claiming assets from the vault will revert for all users, temporarily (and sometimes permanently) locking user funds in the contract.

## Proof of concept

The scenario above is demonstrated in the test, change the following test in test/integration/vault/Vault.test.ts:
```ts
    it('simple deposits and redemptions', async () => {
...
      // Now we should have opened positions.
      // The positions should be equal to (smallDeposit + largeDeposit) * leverage originalOraclePrice.
      expect(await position()).to.equal(
        smallDeposit.add(largeDeposit).mul(leverage).mul(4).div(5).div(originalOraclePrice),
      )
      expect(await btcPosition()).to.equal(
        smallDeposit.add(largeDeposit).mul(leverage).div(5).div(btcOriginalOraclePrice),
      )

      /*** remove all lines after this and replace with the following code: ***/

      var half = smallDeposit.add(largeDeposit).div(2).add(smallDeposit);
      await vault.connect(user).update(user.address, 0, half, 0)

      await updateOracle()
      await vault.connect(user2).update(user2.address, smallDeposit, 0, 0) // this will create min position in the market
      await vault.connect(user).update(user.address, 0, 0, half) // this will revert even though it's just claiming
    })
```

The last line in the test will revert, even though it's just claiming assets. If the pre-last line is commented out (no "min position" created in the market), it will work normally.

## Code Snippet

Ineligable amount calculation adds `withdrawal`:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L431

`withdrawal` is subtracted twice - once directly from collateral, 2nd time via ineligable amount subtractions:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L118-L119

## Tool used

Manual Review

## Recommendation

Remove `add(withdrawal)` from `_ineligable` calculation in the vault.