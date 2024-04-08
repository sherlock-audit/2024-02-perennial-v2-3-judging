Damp Coconut Jaguar

high

# MultiInvoker's stored TriggerOrders are not migrated to new format, potentially causing huge interface fees charged to users.

## Summary

`TriggerOrder` struct which is used to store orders in `MultiInvoker` has changed in v2.3 compared to v2.2, however, there is no migration process for these orders mentioned in the [Migration Guide](https://github.com/equilibria-xyz/perennial-v2/blob/22ba19c323a13c9f02f95db6747d137a3bf1277a/runbooks/MIGRATION_v2.2.md), meaning there might be problems with existing v2.2 orders, in particular:
- previous version's `interfaceFeeAmount` is now in unused area (meaning interfaces won't receive fees for these orders)
- previous version's `interfaceFeeUnwrap` collides with the new version's `interfaceFeeAmount1` (meaning that if existing order had `interfaceFeeUnwrap = true`, the new version will handle it as a 2^40 fee value (in UFixed6 format 2^40 = ~1.1M USDC), charging huge interface fee from unsuspecting users when the order executes). This will likely revert for insufficient funds to pay the fee, but if the user does have 1.1M+ collateral, this huge amount will be charged to interface fee for order exectution.
- new version's `interfaceFeeUnwrap` are in the previously unallocated area, meaning it will be `false` for all previous orders, potentially creating problems/funds loss for interfaces which can't handle DSU token and needed underlying (USDC) instead.

## Vulnerability Detail

Previous version's `TriggerOrder`:
```solidity
struct StoredTriggerOrder {
    /* slot 0 */
    uint8 side;                // 0 = maker, 1 = long, 2 = short, 3 = collateral
    int8 comparison;           // -2 = lt, -1 = lte, 0 = eq, 1 = gte, 2 = gt
    uint64 fee;                // <= 18.44tb
    int64 price;               // <= 9.22t
    int64 delta;               // <= 9.22t
    uint48 interfaceFeeAmount; // <= 281m

    /* slot 1 */
    address interfaceFeeReceiver;
    bool interfaceFeeUnwrap;
    bytes11 __unallocated0__;
}
```

New version's `TriggerOrder`:
```solidity
struct StoredTriggerOrder {
    /* slot 0 */
    uint8 side;         // 0 = maker, 1 = long, 2 = short
    int8 comparison;    // -2 = lt, -1 = lte, 0 = eq, 1 = gte, 2 = gt
    uint64 fee;         // <= 18.44tb
    int64 price;        // <= 9.22t
    int64 delta;        // <= 9.22t
    bytes6 __unallocated0__; // @audit existing orders interfaceFeeAmount will be there

    /* slot 1 */
    address interfaceFeeReceiver1;
    uint48 interfaceFeeAmount1;      // <= 281m @audit existing orders unwrap flag here (in the high order byte), thus either 0 or 2^40 for existing orders
    bool interfaceFeeUnwrap1;   // @audit previously unallocated, thus false for all existing orders
    bytes5 __unallocated1__;

    /* slot 2 */
    address interfaceFeeReceiver2;
    uint48 interfaceFeeAmount2;      // <= 281m
    bool interfaceFeeUnwrap2;
    bytes5 __unallocated2__;
}
```

Notice the @audit remarks in the code. When existing v2.2 orders will be executed in v2.3 code, the overlapping fields in slot1 will cause incorrect values for these orders.

## Impact

For existing orders with interface fee specified:
- if `interfaceFeeUnwrap` was set to `true`, fee charged will be 2^40 (==~1.1M USDC), unexpected fee amount for the user. Additionally, `interfaceFeeUnwrap` will be treated as `false`, thus interface will receive this fee of 2^40 as DSU rather than underlying (USDC).
- if `interfaceFeeUnwrap` was set to `false`, fee charged will be 0, fee funds lost by the interfaceFee account which expected to receive it.

## Code Snippet

Previous version of `TriggerOrder`:
https://github.com/equilibria-xyz/perennial-v2/blob/7e60e69de9a613bfb449dc976801a000daa72aa4/packages/perennial-extensions/contracts/types/TriggerOrder.sol#L18-L31

New version of `TriggerOrder`:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-extensions/contracts/types/TriggerOrder.sol#L19-L39

## Tool used

Manual Review

## Recommendation

Possible solutions:
- refactor new `TriggerOrder` format to avoid fields collision (and carry over all values from existing orders)
- include version id in the struct and read differently depending on version
- clear all pending orders during migration (and let users know beforehand that all orders will be cancelled in the new version)