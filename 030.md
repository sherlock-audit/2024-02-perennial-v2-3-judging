Swift Cornflower Manatee

medium

# after update v2.1.1  may not be able to settle properly

## Summary
Updating `v2.1.1` will set `settleOnly=true` and settle all users.
The latest `OracleVersion` needs to be commited before settle.
However, commit the `OracleVersion` may trigger `market.update()` and prevent the `OracleVersion` from being updated properly.
## Vulnerability Detail

An important step in updating v2.2 is to settle all users first.

>## Upgrade to v2.1.1
>Upgrade the protocol contracts to https://github.com/equilibria-xyz/perennial-v2/pull/257, to add settle only mode.
>
>This is upgrade-safe since the new settleOnly field in the risk parameters defaults to false.
>## Enter settle only mode, and settle all accounts on all markets / vaults
>Next we turn on the settleOnly parameter to true. This pauses all new updates to the markets, while still allowing settlement.
>
>We must then go through and settle every single account that has an unsettled position present in each market. This can be batched via a multicall contract.

1. Upgrade to v2.1.1
2. set `settleOnly=true`  -> will pause all `market.update()`
3. wait last OracleVersion
4. settle all user

We need to wait for the latest `OracleVersion` before settle all users.

But currently, updating the latest `OracleVersion` triggers `market.update()`.
```solidity
contract KeeperOracle is IKeeperOracle, Instance {
...
    function commit(OracleVersion memory version) external onlyFactory returns (bool requested) {
        if (version.timestamp == 0) revert KeeperOracleVersionOutsideRangeError();
        requested = (version.timestamp == next()) ? _commitRequested(version) : _commitUnrequested(version);
        _global.latestVersion = uint64(version.timestamp);

        for (uint256 i; i < _globalCallbacks[version.timestamp].length(); i++)
@>          _settle(IMarket(_globalCallbacks[version.timestamp].at(i)), address(0));

        emit OracleProviderVersionFulfilled(version);
    }


    function _settle(IMarket market, address account) private {
@>      market.update(account, UFixed6Lib.MAX, UFixed6Lib.MAX, UFixed6Lib.MAX, Fixed6Lib.ZERO, false);
    }
```
`market.update()` with `settleOnly=true` is not executable

So it can't update the latest `OracleVersion`, so it can't really `settle`.

It is recommended to use a method dedicated to settlement: `market.settle()`.

## Impact
If all settlements are not made before `settleOnly = true`, or if someone maliciously submits the order before executing `settleOnly = true`
This could lead to not settle properly after updating v2.1.1
## Code Snippet
https://github.com/equilibria-xyz/perennial-v2/blob/22ba19c323a13c9f02f95db6747d137a3bf1277a/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L179
## Tool used

Manual Review

## Recommendation
```diff
contract KeeperOracle is IKeeperOracle, Instance {
...
    function _settle(IMarket market, address account) private {
-       market.update(account, UFixed6Lib.MAX, UFixed6Lib.MAX, UFixed6Lib.MAX, Fixed6Lib.ZERO, false);
+       market.settle(account);
    }
```