Keen Coal Tarantula

medium

# Orders on Optimism chains can not be settled due to revert of ````keep()````

## Summary
After ````Ecotone```` upgrade, Optimism's ````OptGasInfo("0x420000000000000000000000000000000000000F").scalar()```` function has been deprecated, which would cause ````Kept_Optimism```` contract  ````revert```` always.

Reference: https://docs.optimism.io/stack/transactions/fees#l1-data-fee

## Vulnerability Detail
The issue arises on L29 of ````_calldataFee()```` function, as ```` OPT_GAS.scalar()```` would revert.

```solidity
File: contracts\attribute\Kept\Kept_Optimism.sol
14: abstract contract Kept_Optimism is Kept {
...
16:     OptGasInfo constant OPT_GAS = OptGasInfo(0x420000000000000000000000000000000000000F);

20:     function _calldataFee(
...
24:     ) internal view virtual override returns (UFixed18) {
25:         return _fee(
26:             OPT_GAS.getL1GasUsed(applicableCalldata),
27:             multiplierCalldata,
28:             bufferCalldata,
29:             OPT_GAS.l1BaseFee() * OPT_GAS.scalar() / (10 ** OPT_GAS.decimals()) // @audit revert due to OPT_GAS.scalar()
30:         );
31:     }
32: }

```

The following PoC is built on both Optimism  and Base mainnet, we can see ```` OPT_GAS.scalar()```` reverts with ````GasPriceOracle: scalar() is deprecated```` message.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

interface OptGasInfo {
    function scalar() external view returns (uint256);
}

contract OptimismKeepFeeBug is Test {
    uint256 constant BLOCK_HEIGHT = 12520000;//118120000; // Mar-30-2024 10:46:17 PM +UTC
    string constant RPC_URL = "https://mainnet.base.org";//"https://mainnet.optimism.io";
    OptGasInfo constant OPT_GAS = OptGasInfo(0x420000000000000000000000000000000000000F);

    function setUp() public {
        vm.createSelectFork(RPC_URL, BLOCK_HEIGHT);
    }

    function testOptGasInfoScalarCallRevert() public {
        vm.expectRevert();
        OPT_GAS.scalar();
    }
}
```

the test log:
```solidity
2024-02-perennial-v2-3\root> forge test --mc OptimismKeepFeeBug -vvvv
[⠒] Compiling...
[⠊] Compiling 1 files with 0.8.23Compiler run successful!
[⠒] Compiling 1 files with 0.8.23
[⠢] Solc 0.8.23 finished in 2.48s

Running 1 test for test/OptimismKeepFeeBug.t.sol:OptimismKeepFeeBug
[PASS] testOptGasInfoScalarCallRevert() (gas: 13337)
Traces:
  [13337] OptimismKeepFeeBug::testOptGasInfoScalarCallRevert()
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← ()
    ├─ [7434] 0x420000000000000000000000000000000000000F::scalar() [staticcall]
    │   ├─ [2436] 0xb528D11cC114E026F138fE568744c6D45ce6Da7A::scalar() [delegatecall]
    │   │   └─ ← revert: GasPriceOracle: scalar() is deprecated
    │   └─ ← revert: GasPriceOracle: scalar() is deprecated
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 842.07ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## Impact
Orders can not be settled, break of core functionality.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/root/contracts/attribute/Kept/Kept_Optimism.sol#L29

## Tool used

Manual Review

## Recommendation
reference: https://docs.optimism.io/stack/transactions/fees#ecotone