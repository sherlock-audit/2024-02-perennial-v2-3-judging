Formal White Bull

medium

# Vault Rounding Issue in `Vault.sol::_update`

## Summary
 according [EIP4626 Rounding Specification](https://eips.ethereum.org/EIPS/eip-4626) the [`claimAmount`](https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L308) should be rounded up.

## Vulnerability Detail
this protocol rounds down the `claimAmount` by default .t
According to the EIP4626 rounding specification the rounding of the `claimAmount` must be up but the `_socialize` function which returns the `claimAmount` doesn't take into condition that constraint and just runs it down by default.

## Impact

this current implementation will round down the `claimAmount` expected instead of otherwise.

## Code Snippet
`claimAmount`
```UFixed6 claimAmount = _socialize(context, claimAssets);```

`Vault.sol::_socialize`
```solidity
    function _socialize(Context memory context, UFixed6 claimAssets) private pure returns (UFixed6) {
        return context.global.assets.isZero() ?
            UFixed6Lib.ZERO :
            claimAssets.muldiv(
                UFixed6Lib.unsafeFrom(context.totalCollateral).min(context.global.assets),
                context.global.assets
            );
    }
```

## Tool used

Manual Review

## Recommendation
the implementation of `_socialize` should be checked to make sure `claimAmount` will have a rounded up value