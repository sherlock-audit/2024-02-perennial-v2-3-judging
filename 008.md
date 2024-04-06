Damp Coconut Jaguar

medium

# When vault's market weight is set to 0 to remove the market from the vault, vault's leverage in this market is immediately set to max leverage risking position liquidation

## Summary

If any market has to be removed from the vault, the only way to do this is via setting this market's weight to 0. The problem is that the first vault rebalance will immediately withdraw max possible collateral from this market, leaving vault's leverage at max possible leverage, risking the vault's position liquidation. This is especially dangerous if vault's position in this removed market can not be closed due to high skew, so min position is not 0, but the leverage will be at max possible value. As a result, vault depositors can lose funds due to liquidation of vault's position in this market.

## Vulnerability Detail

When vault is rebalanced, each market's collateral is calculated as following:
```solidity
    marketCollateral = marketContext.margin
        .add(collateral.sub(totalMargin).mul(marketContext.registration.weight));

    UFixed6 marketAssets = assets
        .mul(marketContext.registration.weight)
        .min(marketCollateral.mul(LEVERAGE_BUFFER));
```

For removed markets (`weight = 0`), `marketCollateral` will be set to `marketContext.margin` (i.e. minimum valid collateral to have position at max leverage), `marketAssets` will be set to 0. But later the position will be adjusted in case minPosition is not 0:
```solidity
    target.position = marketAssets
        .muldiv(marketContext.registration.leverage, marketContext.latestPrice.abs())
        .max(marketContext.minPosition)
        .min(marketContext.maxPosition);
```

This means that vault's position in the market with weight 0 will be at max leverage until liquidated or position can be closed.

## Impact

Market removed from the vault (weight set to 0) is put at max leverage and has a high risk of being liquidated, thus losing vault depositors funds.

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

      console.log("pos1 = " + (await position()) + " pos2 = " + (await btcPosition()) + " col1 = " + (await collateralInVault()) + " col2 = " + (await btcCollateralInVault()));

      // update weight
      await vault.connect(owner).updateWeights([parse6decimal('1.0'), parse6decimal('0')])

      // do small withdrawal to trigger rebalance
      await vault.connect(user).update(user.address, 0, smallDeposit, 0)
      await updateOracle()

      console.log("pos1 = " + (await position()) + " pos2 = " + (await btcPosition()) + " col1 = " + (await collateralInVault()) + " col2 = " + (await btcCollateralInVault()));
    })
```

Console log:
```solidity
pos1 = 12224846 pos2 = 206187 col1 = 8008000000 col2 = 2002000000
pos1 = 12224846 pos2 = 206187 col1 = 9209203452 col2 = 800796548
```

Notice, that after rebalance, position in the removed market (pos2) is still the same, but the collateral (col2) reduced to minimum allowed.

## Code Snippet

Vault market allocation sets collateral to only the margin if `weight = 0`:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L152-L153

## Tool used

Manual Review

## Recommendation

Ensure that the market's collateral is based on leverage even if `weight = 0`