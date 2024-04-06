Keen Coal Tarantula

medium

# ````Pyth```` oracle is paying ````30%```` more than intended keep fee

## Summary
After Pyth network's ````perseus```` upgrade (````Sep-20-2023````), the gas cost of price submission has dramatically decreased. Perennial V2 is paying 30% more than intended keeper fee due to lack of adaptation with the upgrade.

Reference: https://pyth.network/blog/perseus-network-upgrade

## Vulnerability Detail
A key improvement of ````perseus```` upgrade is that the VAA length has significantly decreased, it reduces both L1 calldata fee and L2 execution fee. As most cases L1 calldata fee is the dominant part of gas cost for L2 transactions, the following PoC shows the difference between actual calldata gas used and the amount Perennial V2 is paying. We can see the current implementation is tuned for the old VAA format with about ````6%```` extra compensation for keepers. But for the new VAA format, the compensation increases to ````36%````, there is a ````30%```` over payment than intended.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "forge-std/StdAssertions.sol";
import "contracts/attribute/interfaces/IKept.sol";

interface IPythFactory is IKept {
    function commit(bytes32[] memory ids, uint256 version, bytes calldata data) external;
    function commitKeepConfig(uint256 numRequested) external view returns (KeepConfig memory);
}

interface IPyth {
    struct Price {
        int64 price;
        uint64 conf;
        int32 expo;
        uint publishTime;
    }

    struct PriceFeed {
        bytes32 id;
        Price price;
        Price emaPrice;
    }

    function parsePriceFeedUpdates(
        bytes[] calldata updateData,
        bytes32[] calldata priceIds,
        uint64 minPublishTime,
        uint64 maxPublishTime
    ) external payable returns (PriceFeed[] memory priceFeeds);
}

contract PythOracleL1CalldataFeeTest is Test {
    uint256 constant ARB_GAS_MULTIPLIER = 16;
    uint256 constant ARB_FIXED_OVERHEAD = 140;
    bytes32 constant PYTH_ETH_FEED = 
        hex"ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace";
    IPyth constant PYTH = IPyth(0xff1a0f4744e8582DF1aE09D5611b887B6a12925C);
    IPythFactory constant PYTH_ORACLE_FACTORY = 
        IPythFactory(0x6b60e7c96B4d11A63891F249eA826f8a73Ef4E6E);
    uint256 constant BLOCK_HEIGHT = 196472000; // Apr-01-2024 12:50:35 PM +UTC
    string constant RPC_URL = "https://arbitrum.blockpi.network/v1/rpc/public";

    function setUp() public {
        vm.createSelectFork(RPC_URL, BLOCK_HEIGHT);
    }

    function test() public {
        // 1. check vaa valid
        _checkVaaValid(_oldVaaSample());
        _checkVaaValid(_newVaaSample());

        // 2. L1 calldata gas needed for submitting old VAA
        uint256 oldVaaGas = _calldataGasActuallyUsed(_oldVaaSample());

        // 3. L1 calldata gas needed for submitting new VAA
        uint256 newVaaGas = _calldataGasActuallyUsed(_newVaaSample());

        // 4. L1 calldata gas used for calculating keep fee payment
        IKept.KeepConfig memory config = PYTH_ORACLE_FACTORY.commitKeepConfig(1);
        uint256 payingGas = _calldataGasForPaying(config);

        // 5. show over payment infomation
        _showOverPaymentInfo(oldVaaGas, payingGas, "Old VAA L1 calldata gas info: ");
        _showOverPaymentInfo(newVaaGas, payingGas, "New VAA L1 calldata gas info: ");
    }

    function _checkVaaValid(bytes memory vaa) internal {
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = vaa;
        bytes32[] memory priceIds = new bytes32[](1);
        priceIds[0] = PYTH_ETH_FEED;
        IPyth.PriceFeed[] memory parsedPrices = PYTH
            .parsePriceFeedUpdates{value: 1e9}(updateData, priceIds, 0, type(uint64).max);
        assertEq(1, parsedPrices.length);
        IPyth.PriceFeed memory p = parsedPrices[0];
        assertEq(PYTH_ETH_FEED, p.id);
        assertGt(p.price.price, 0);
        assertEq(p.price.expo, -8);
        assertGt(p.price.publishTime, 0);
    }

    function _calldataGasActuallyUsed(bytes memory vaa) internal view returns(uint256) {
        bytes32[] memory ids = new bytes32[](1);
        ids[0] = PYTH_ETH_FEED;
        uint256 version = block.timestamp;
        bytes memory commitCalldata = abi.encodeCall(IPythFactory.commit, (ids, version, vaa));
        return (commitCalldata.length + ARB_FIXED_OVERHEAD) * ARB_GAS_MULTIPLIER;
    }

    function _calldataGasForPaying(IKept.KeepConfig memory config) internal pure returns(uint256) {
        // reference: Kept_Arbitrum._calldataFee()
        bytes memory applicableCalldata = msg.data[0:0]; // reference: KeeperFactory.commit(), L188
        uint256 applicableGas = ARB_GAS_MULTIPLIER * (applicableCalldata.length + ARB_FIXED_OVERHEAD);
        UFixed18 multiplier = config.multiplierCalldata;
        uint256 buffer = config.bufferCalldata;
        UFixed18 totalGas = UFixed18Lib.from(applicableGas).mul(multiplier).add(UFixed18Lib.from(buffer));
        return totalGas.truncate();
    }

    function _newVaaSample() internal pure returns(bytes memory vaa) {
        vaa = hex"504E41550100000003B801000000030D0160E76CBD81000E91C8FEB71136E4CB10B5A17725516681715D402FB79D9987672D0A0EC40A6AB1E98452431170042DFFC4FF38FD7F5D4522868B635E4FFDBCE20102CB35B4C97B52B40DCD3E477C684EC17249AE29A305EEF9615DBA8922C38298C870367F01EF0ED9D18C5627EAEAAFEC089515FFEA9BF7501FF85966C3A798EA4100045047AFCACDDC270F931EA07CAE16F5477D5FC28DF3C0A9CA47D890B1FFD9AFF666B35D80D3B64E402C0F9F248775064A4DD67785EE42EE9B57741F56220AC88900060976787906401E57FEA2007F7BF8DFB3EFB41DE4D6BF269C1136C07F8CD7A932513D4EDB784DF937733D6C1712D58BA62BBBB2029BB4C4CC8B768FBA517B0EE700077E770DF0D8528FEB38B3EADF45602DB1DA5704EF97701FFC96130ECC904A26821DDA5AA0554FE8C8C50A84008237B26DC26F6E9D4A07C8CCA25767287F4932550108F1AB8D73A9805E4D69A048431950F3AD046AF550D39C2FE490C0A2BEC4C45CF039D470299725AAA88D7CB88E01165C864D5E10487EF86B7BBE71E5F19D89A2AF010A0A4131D585AD52890BB642F3950684548277368ACB54F47AC252301E70BBDE8362A24B19976D0D23CF3DD396614A70AF285B7FFE7D83CD378382B0D85376645E000BB5F9ECC068258CB009608DF75A0BA8D20AFB144100172A53103A4140EE7AC8FF6C58A37AC553A32B2E45A702C323E019EC03AB240F0167C6C76B4A3D4B96AB41010CFAC6E0EC1CCCFC26B137CCDED3A89216D7833E7F4F0E4A58BA15247D21EB902D09ECDC331F55BB228CE592488D6658FDFEFE1E78CDC05E3113797CBEAEE4DA75000DFB4D81515D7D5A37C125628D808CEDA74F1A378467D83C8D17F367AD1465AACF7B8022C86179EBB271A56C81A624DBD9C86340ADAA3D4B7908C3BFDDFC635AB4010E73ACAF84B1E0FB9BBF8398AC0F4F6D417494EDA370100EF6DF874C579657F2475FF6EA8E8B6F4F55B25CC959EECA2DF16EDEC870B5738F4535421F10B1065C0F01100F698DBC6D22A9A2EF0F8C98B5C23E2AB300AD137A9BAD7350BA2A3D1033104B578B5BA36FE88EDD33A3E1224A5B840A801AB0A049CD4108B21823A3442E71D70112323CD4B363F2CB5722C165101E089D398636CE2CB60F8EC0EDDFFBA792B8919474155A5EAE3045C5436B3DEFC162123FCB4EE169BD51A92B9130242EF48585830065FB3C1500000000001AE101FAEDAC5851E32B9B23B5F9411A8C2BAC4AAE3ED4DD7B811DD1A72EA4AA710000000002BDDD030141555756000000000007C72D800000271039CE17BFF50C540B7C31E6E82634F845D1926C6001005500FF61491A931112DDF1BD8147CD1B641375F79F5825126D665480874634FD0ACE0000004E7EDF1C00000000000C751640FFFFFFF80000000065FB3C150000000065FB3C140000004DBE57B1E000000000115376D40AAB92497D68C9A018BCF9EFD25191A3EA85FE4238CE5B1CBA37F8C61D529099CEFDAC3741E9E9AC6DD9BBC1BE085597D4AA4C4D3587CDBE332590B2A4F4585F9B8F35D553EDBE7AA698956D981E4D67303487809B33E9E2CE8A95202691A0B7F0A7EA9AF8B0FEA8B9866892CFF27F985107CD285C352A8E9A31AF638A20A57715381173E54660C662DD2E4124D22C1237427BC2C87EDF166BC81AD75F429567F33B8E5848A09E98F66EEB41D601DCD5AE578C4352F01C88C4DD284BF1BE6CF9958F66A5701BEA1E61";
    }

    function _oldVaaSample() internal pure returns(bytes memory vaa) {
        // reference: Pyth.test.ts, L35
        vaa = hex"01000000030d0046d9570837d4d2cfcc50fd3346bf18df10179011a1e125e27467bb47bc26f8ce194ed59b48793f4dec9dc919f2e22d43e33fe6ac263980da08f502b894e0fe6f00026a8d75df8f46b144e48ebf6bd6a63267e90dafe353021bbecb9909b0bef0434e56f696870e41ce16b9b8b40c22548815c5fe157915cd08366cb439c5eb51029001040c1775df612c74d4e18da4daa0f42b8953d92340aadf757b6b6ee1e549d1ddfe0115628a360168a23a05684afa5389dd75c431eb5228aaa528de450aae38a50f01084aeacdb58103612ab202ac887d53dc14cd10c4e7d788f95685e0944fb919f8ec64b5cdaa4ede600a9f89ed9aaa3be86facac6ede2ed08760101f6c5c23ce6b29010a2847bd95a0dd2d14302ee732f8f3547ea7e1bfcc9f258ab07d33ca9c62fc837621e8a7dcdb41b06db6f8e768e7e5510f3954029fcdf6e8f3d6b4072da73b51ae010b3a693388722a579f8e7ce44bceb0fac79a4dffbd1076b99a79c55286cc2cf28f2feda95aaf1823f6da2922d9f675619931107bd0538e9dbd53025463a95f2b7b010c732680bb2ba4843b67ba4c493d29cbfe737729cb872aec4ac9b8d83eb0fec898556d02bdeae8995870dc6e75187feacc9b9f714ddd9d97ba5a5abbe07d8884f2010debe8a41fe1715b27fbf2aba19e9564bb4e0bde1fc29412c69347950d216a201130e301f43a5aeec8e7464c9839a114e22efe65d49128b4908b9fa618476cc519010e33495ea1a8df32bc3e7e6f353a4d0371e8d5538e33e354e56784e2877f3765ef5e774abb0c50973686f8236adf5979225ff6f6c68ed942197f40c4fed59331bc010fead2505d4be9161ab5a8da9ed0718afd1ddf0b7905db57997a1ed4741d9d326840e193b84e115eba6256ed910e12e10f68c4563b6abaae211eaac5c0416d1f9601108eddcab6c9952dc0da91900a35821ef75818a5f3898721fd05ff708597c19d5e573f2b63674989365ca9fee0dd660566afaec135230d978e66ee4106c263b124011164788fde3bcf11e6773308a3732a0f0bd73b6876789c2b01f2bbaf84473be6ec2b7a3884d117adc625cbf48710c238d9c122a5f64f283685d9c66f3656d79d4d001247f246ba17092100f8bfc1e93822ad3d07561697ac90d4ebf3d371fce17e399246b18f85b52f74157240cdf16da4bde72146cf0cb976c39a2d6958d7b55773f70064815acc00000000001af8cd23c2ab91237730770bbea08d61005cdda0984348f3f6eecb559638c0bba0000000001b8413740150325748000300010001020005009d04028fba493a357ecde648d51375a445ce1cb9681da1ea11e562b53522a5d3877f981f906d7cfe93f618804f1de89e0199ead306edc022d3230b3e8305f391b00000002aa3fa23ae00000000117b5092fffffff80000002a9cdd1528000000000f4ab712010000000a0000000c0000000064815acc0000000064815acb0000000064815acb0000002aa3fa23ae00000000117b50920000000064815acbe6c020c1a15366b779a8c870e065023657c88c82b82d58a9fe856896a4034b0415ecddd26d49e1a8f1de9376ebebc03916ede873447c1255d2d5891b92ce57170000002c5ffd594000000000086bfba4fffffff80000002c55aaa600000000000a73c7010100000007000000080000000064815acc0000000064815acb0000000064815acb0000002c5ffd594000000000086bfba40000000064815acbc67940be40e0cc7ffaa1acb08ee3fab30955a197da1ec297ab133d4d43d86ee6ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace0000002acc544cc90000000006747a77fffffff80000002ac4a4eba8000000000456d9f30100000017000000200000000064815acc0000000064815acb0000000064815acb0000002acc544cc90000000006747a770000000064815acb8d7c0971128e8a4764e757dedb32243ed799571706af3a68ab6a75479ea524ff846ae1bdb6300b817cee5fdee2a6da192775030db5615b94a465f53bd40850b50000002ac880a3200000000010e67139fffffff80000002abc130ec0000000001b3bcc6401000000090000000a0000000064815acc0000000064815acb0000000064815acb0000002ac880a3200000000010e671390000000064815acb543b71a4c292744d3fcf814a2ccda6f7c00f283d457f83aa73c41e9defae034ba0255134973f4fdf2f8f7808354274a3b1ebc6ee438be898d045e8b56ba1fe1300000000000000000000000000000000fffffff8000000000000000000000000000000000000000000000000080000000064815acc0000000064815aca0000000000000000000000000000000000000000000000000000000000000000";
    }

    function _showOverPaymentInfo(uint256 used, uint256 paid, string memory prefix) internal pure {
        uint256 diff = paid - used;
        string memory usedInfo = string.concat("used(", _toDigitalsString(used), "), ");
        string memory paidInfo = string.concat("paid(", _toDigitalsString(paid), "), ");
        string memory overPaymentInfo = 
            string.concat("over payment ratio (", _toPercentString(diff * 10000 / used), ")");
        string memory allInfo = string.concat(prefix, usedInfo, paidInfo, overPaymentInfo);
        console2.log(allInfo);
    }

    function _toPercentString(uint256 pct) internal pure returns (string memory result) {
        uint256 d2 = pct % 10;
        uint256 d1 = (pct / 10) % 10;
        uint256 d0 = (pct / 100) % 10;
        result = string.concat(_toString(d0), ".", _toString(d1), _toString(d2), "%");
        uint256 d = pct / 1000;
        while (d > 0) {
            result = string.concat(_toString(d % 10), result);
            d = d / 10;
        }
    }

    function _toDigitalsString(uint256 num) internal pure returns(string memory result) {
        while (num > 0) {
            result = string.concat(_toString(num % 10), result);
            num /= 10;
        }
        if (bytes(result).length == 0) result = "0";
    }


    function _toString(uint256 digital) internal pure returns (string memory str) {
        str = new string(1);
        bytes16 symbols = "0123456789abcdef";
        assembly {
            mstore8(add(str, 32), byte(digital, symbols))
        }
    }
}

```


the test log:
```solidity
2024-02-perennial-v2-3\root> forge test --mc PythOracleL1CalldataFeeTest -vv
[⠰] Compiling...
[⠆] Compiling 1 files with 0.8.23
[⠰] Solc 0.8.23 finished in 2.84s
Compiler run successful!

Ran 1 test for test/PythOracleL1CalldataFee.t.sol:PythOracleL1CalldataFeeTest
[PASS] test() (gas: 310418)
Logs:
  Old VAA L1 calldata gas info: used(33024), paid(35200), over payment ratio (6.58%)
  New VAA L1 calldata gas info: used(25856), paid(35200), over payment ratio (36.13%)

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 970.84ms (41.09ms CPU time)

Ran 1 test suite in 979.14ms (970.84ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## Impact
The protocol continuously loses fund.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/root/contracts/attribute/Kept/Kept_Arbitrum.sol#L21

## Tool used

Manual Review

## Recommendation
Decreasing L1 calldata and L2 execution keeper fee based on the new VAA format.