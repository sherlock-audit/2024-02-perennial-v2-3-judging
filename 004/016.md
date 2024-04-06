Damp Coconut Jaguar

medium

# If referral or liquidator is the same address as the account, then liquidation/referral fees will be lost due to local storage being overwritten after the `claimable` amount is credited to liquidator or referral

## Summary

Any user (address) can be liquidator and/or referral, including account's own address (the user can self-liquidate or self-refer). During the market settlement, liquidator and referral fees are credited to liquidator/referral's `local.claimable` storage. The issue is that the account's local storage is held in the memory during the settlement process, and is saved into storage after settlement/update. This means that `local.claimable` storage changes for the account are not reflected in the in-memory cached copy and discarded when the cached copy is saved after settlement.

This leads to liquidator and referral fees being lost when these are the account's own address.

## Vulnerability Detail

During market account settlement process, in the `_processOrderLocal`, liquidator and referral fees are credited to corresponding accounts via:
```solidity
...
    _credit(liquidators[account][newOrderId], accumulationResult.liquidationFee);
    _credit(referrers[account][newOrderId], accumulationResult.subtractiveFee);
...
function _credit(address account, UFixed6 amount) private {
    if (amount.isZero()) return;

    Local memory newLocal = _locals[account].read();
    newLocal.credit(amount);
    _locals[account].store(newLocal);
}
```

However, for the account the cached copy of `_locals[account]` is stored after the settlement in `_storeContext`:
```solidity
function _storeContext(Context memory context, address account) private {
    // state
    _global.store(context.global);
    _locals[account].store(context.local);
...
```

The order of these actions is:
```solidity
function settle(address account) external nonReentrant whenNotPaused {
    Context memory context = _loadContext(account);

    _settle(context, account);

    _storeContext(context, account);
}
```

1. Load `_locals[account]` into memory (`context.local`)
2. Settle: during settlement `_locals[account].claimable` is increased for liquidator and referral. Note: this is not reflected in `context.local`
3. Store cached context: `_locals[account]` is overwritten with the `context.local`, losing `claimable` increased during settlement.

## Impact

If user self-liquidates or self-refers, the liquidation and referral fees are lost by the user (and are stuck in the contract, because they're still subtracted from the user's collateral).

## Proof of concept

The scenario above is demonstrated in the test, add this to test/unit/market/Market.test.ts:
```ts
it('self-liquidation fees lost', async () => {
const POSITION = parse6decimal('100.000')
const COLLATERAL = parse6decimal('120')

function setupOracle(price: string, timestamp : number, nextTimestamp : number) {
    const oracleVersion = {
    price: parse6decimal(price),
    timestamp: timestamp,
    valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion.timestamp).returns(oracleVersion)
    oracle.status.returns([oracleVersion, nextTimestamp])
    oracle.request.returns()
}

dsu.transferFrom.whenCalledWith(user.address, market.address, COLLATERAL.mul(1e12)).returns(true)
dsu.transferFrom.whenCalledWith(userB.address, market.address, COLLATERAL.mul(1e12)).returns(true)

var time = TIMESTAMP;

setupOracle('1', time, time + 100);
await market.connect(user)
    ['update(address,uint256,uint256,uint256,int256,bool)'](user.address, POSITION, 0, 0, COLLATERAL, false);

time += 100;
setupOracle('1', time, time + 100);
await market.connect(userB)
    ['update(address,uint256,uint256,uint256,int256,bool)'](userB.address, 0, POSITION, 0, COLLATERAL, false);

time += 100;
setupOracle('1', time, time + 100);

time += 100;
setupOracle('0.7', time, time + 100);

// self-liquidate
setupOracle('0.7', time, time + 100);
await market.connect(userB)
    ['update(address,uint256,uint256,uint256,int256,bool)'](userB.address, 0, 0, 0, 0, true);

// settle liquidation
time += 100;
setupOracle('0.7', time, time + 100);
await market.settle(userB.address);
var info = await market.locals(userB.address);
console.log("Claimable userB: " + info.claimable);
```

Console log:
```solidity
Claimable userB: 0
```

## Code Snippet

`Market._credit` modifies `local.claimable` storage for the account:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L678-L684

## Tool used

Manual Review

## Recommendation

Modify `Market._credit` function to increase `context.local.claimable` if account to be credited matches account which is being updated.