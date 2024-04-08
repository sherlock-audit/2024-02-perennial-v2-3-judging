Damp Coconut Jaguar

medium

# Makers can lose funds from price movement even when no long and short positions are opened, due to incorrect distribution of adiabatic fees exposure between makers

## Summary

Adiabatic fees introduced in this new update of the protocol (v2.3) were introduced to solve the problem of adiabatic fees netting out to 0 in market token's rather than in USD terms. With the new versions, this problem is solved and adiabatic fees now net out to 0 in USD terms. However, they net out to 0 only for the whole makers pool, but each individual maker can have profit or loss from adiabatic fees at different price levels all else being equal. This creates unexpected risk of loss of funds from adiabatic fees for individual makers, which can be significant, up to several percents of the amount invested.

## Vulnerability Detail

The issue is demonstrated in the following scenario:
- price = 1
- Alice open `maker = 10` (`collateral = +0.9` from adiabatic fee)
- Bob opens `maker = 10` (`collateral = +0.7` from adiabatic fee)
- Path A. `price = 1`. Bob closes (final collateral = +0), Alice closes (final collaterral = +0)
- Path B. `price = 2`. Bob closes (final collateral = +0.1), Alice closes (final collaterral = -0.1)
- Path C. `price = 0.5`. Bob closes (final collateral = -0.05), Alice closes (final collateral = +0.05)

Notice that both Alice and Bob are the only makers, there are 0 longs and 0 shorts, but still both Alice and Bob pnl depends on the market price due to pnl from adiabatic fees. Adiabatic fees net out to 0 for all makers aggregated (Alice + Bob), but not for individual makers. Individual makers pnl from adiabatic fees is more or less random depending on the other makers who have opened.

If Alice were the only maker, then:
- price = 1
- Alice opens `maker = 10` (`collateral = +0.9`)
- price = 2: exposure adjusted +0.9 (Alice `collateral = +1.8`)
- Alice closes `maker = 10` (adiabatic fees = `-1.8`, Alice final collateral = 0)

For the lone maker there is no such problem, final collateral is 0 regardless of price. The core of the issue lies in the fact that the maker's adiabatic fees exposure adjustment is weighted by makers open maker amount. So in the first example:
- price = 1. Alice `maker = 10, exposure = +0.9`, Bob `maker = 10, exposure = +0.7`
- price = 2. Total exposure is adjusted by +1.6, split evenly between Alice and Bob (+0.8 for each)
- Alice new exposure = 0.9 + 0.8 = +1.7 (but adiabatic fees paid to close = -1.8)
- Bob new exposure = 0.7 + 0.8 = +1.5 (but adiabatic fees paid to close = -1.4)

If maker exposure adjustment was weighted by individual makers exposure, then all is correct:
- price = 1. Alice `maker = 10, exposure = +0.9`, Bob `maker = 10, exposure = +0.7`
- price = 2. Total exposure is adjusted by +1.6, split 0.9:0.7 between Alice and Bob, e.g. +0.9 for Alice, +0.7 for Bob
- Alice new exposure = 0.9 + 0.9 = +1.8 (adiabatic fees paid to close = -1.8, net out to 0)
- Bob new exposure = 0.7 + 0.7 = +1.4 (adiabatic fees paid to close = -1.4, net out to 0)

In the worst case, in the example above, if Bob opens `maker = 40` (adiabatic fees `scale = 50`), then at `price = 2`, Alice's final collateral is `-0.4` due to adiabatic fees. Given that Alice's position is 10 at `price = 2` (`notional = 20`), a loss of `-0.4` is a loss of `-2%` at 1x leverage, which is quite significant.

## Impact

Individual makers bear an additional undocumented price risk due to adiabatic fees, which is quite significant (can be several percentages of the notional).

## Proof of concept

The scenario above is demonstrated in the test, change the following test in test/unit/market/Market.test.ts:
```ts
it('adiabatic fee', async () => {
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

  async function showInfo() {
    await market.settle(user.address);
    await market.settle(userB.address);
    await market.settle(userC.address);
    var sum : BigNumber = BigNumber.from('0');
    var info = await market.locals(user.address);
    console.log("user collateral = " + info.collateral);
    sum = sum.add(info.collateral);
    var info = await market.locals(userB.address);
    sum = sum.add(info.collateral);
    console.log("userB collateral = " + info.collateral);
    var info = await market.locals(userC.address);
    sum = sum.add(info.collateral);
  }

  async function showVer(ver : number) {
    var v = await market.versions(ver);
    console.log("ver" + ver + ": makerValue=" + v.makerValue + " longValue=" + v.longValue + 
    " makerPosFee=" + v.makerPosFee + " makerNegFee=" + v.makerNegFee +
    " takerPosFee=" + v.takerPosFee + " takerNegFee=" + v.takerNegFee
    );
  }

  const riskParameter = { ...(await market.riskParameter()) }
  const riskParameterMakerFee = { ...riskParameter.makerFee }
  riskParameterMakerFee.linearFee = parse6decimal('0.00')
  riskParameterMakerFee.proportionalFee = parse6decimal('0.00')
  riskParameterMakerFee.adiabaticFee = parse6decimal('0.01')
  riskParameterMakerFee.scale = parse6decimal('50.0')
  riskParameter.makerFee = riskParameterMakerFee
  const riskParameterTakerFee = { ...riskParameter.takerFee }
  riskParameterTakerFee.linearFee = parse6decimal('0.00')
  riskParameterTakerFee.proportionalFee = parse6decimal('0.00')
  riskParameterTakerFee.adiabaticFee = parse6decimal('0.01')
  riskParameterTakerFee.scale = parse6decimal('50.0')
  riskParameter.takerFee = riskParameterTakerFee
  await market.connect(owner).updateRiskParameter(riskParameter)

  marketParameter = {
    fundingFee: parse6decimal('0.0'),
    interestFee: parse6decimal('0.0'),
    oracleFee: parse6decimal('0.0'),
    riskFee: parse6decimal('0.0'),
    positionFee: parse6decimal('0.0'),
    maxPendingGlobal: 5,
    maxPendingLocal: 3,
    settlementFee: 0,
    makerCloseAlways: false,
    takerCloseAlways: false,
    closed: false,
    settle: false,
  }
  await market.connect(owner).updateParameter(beneficiary.address, coordinator.address, marketParameter)

  var time = TIMESTAMP;

  setupOracle('1', time, time + 100);
  await market.connect(user)
      ['update(address,uint256,uint256,uint256,int256,bool)'](user.address, POSITION, 0, 0, COLLATERAL, false);
  await showInfo()
  await showVer(time)

  time += 100;
  setupOracle('1', time, time + 100);
  await market.connect(userB)
      ['update(address,uint256,uint256,uint256,int256,bool)'](userB.address, POSITION, 0, 0, COLLATERAL, false);
  await showInfo()
  await showVer(time)

  time += 100;
  setupOracle('1', time, time + 100);
  await showInfo()
  await showVer(time)

  time += 100;
  setupOracle('2', time, time + 100);
  await market.connect(userB)
      ['update(address,uint256,uint256,uint256,int256,bool)'](userB.address, 0, 0, 0, 0, false);
  await showInfo()
  await showVer(time)

  time += 100;
  setupOracle('2', time, time + 100);
  await market.connect(user)
      ['update(address,uint256,uint256,uint256,int256,bool)'](user.address, 0, 0, 0, 0, false);
  await showInfo()
  await showVer(time)

  time += 100;
  setupOracle('0.5', time, time + 100);
  await showInfo()
  await showVer(time)
})
```

Console log:
```solidity
user collateral = 10000000000
userB collateral = 0
ver1636401093: makerValue=0 longValue=0 makerPosFee=0 makerNegFee=0 takerPosFee=0 takerNegFee=0
user collateral = 10000090000
userB collateral = 10000000000
ver1636401193: makerValue=0 longValue=0 makerPosFee=9000 makerNegFee=0 takerPosFee=0 takerNegFee=0
user collateral = 10000090000
userB collateral = 10000070000
ver1636401293: makerValue=0 longValue=0 makerPosFee=7000 makerNegFee=0 takerPosFee=0 takerNegFee=0
user collateral = 10000170000
userB collateral = 10000150000
ver1636401393: makerValue=8000 longValue=0 makerPosFee=0 makerNegFee=0 takerPosFee=0 takerNegFee=0
user collateral = 10000170000
userB collateral = 10000010000
ver1636401493: makerValue=8000 longValue=0 makerPosFee=0 makerNegFee=-14000 takerPosFee=0 takerNegFee=0
user collateral = 9999990000
userB collateral = 10000010000
ver1636401593: makerValue=-5500 longValue=0 makerPosFee=0 makerNegFee=-4500 takerPosFee=0 takerNegFee=0
```

Notice, that final user balance is -0.1 and final userB balance is +0.1

## Code Snippet

Maker exposure is applied to `makerValue`, meaning it's weighted by maker position size:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial/contracts/libs/VersionLib.sol#L314

## Tool used

Manual Review

## Recommendation

Split the total maker exposure by individual maker's exposure rather than by their position size. To do this:
- Add another accumulator to track total exposure
- Add individual maker `exposure` to user's `Local` storage
- When accumulating local storage in the checkpoint, account global accumulator exposure weighted by individual user's exposure.
