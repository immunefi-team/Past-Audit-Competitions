# 28791 - \[SC - Low] The system protects from any rounding issues wh...

Submitted on Feb 27th 2024 at 14:19:55 UTC by @Stormy for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28791

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/LiquidationLibrary.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The system may issue surplus in recovery mode when liquidating cdps with ICR < MCR duo to rounding error.

## Vulnerability Details

On short explanation the liquidating system in eBTC works on the following bases.

* The system is in normal mode as a result any cdps with ICR < MCR can be liquidated.
* The system is in recovery mode duo to that cdps with ICR < CCR can be liquidated.

### Normal mode liquidation

If we look at the function \_liquidateIndividualCdpSetupCDPInNormalMode which liquidates a cdp position with collateral ratio below the minimum one, we can see that the system doesn't allow any spare collateral to be send to the cdp owner when accounting the surplus of the liquidation.

* This is made in case any rounding errors occur when calculating the incentive collateral, as based on the system rules when liquidation happens with a cdp's ICR < MCR the whole cdp collateral should be send to the liquidator.

```solidity
            if (_collSurplus > 0) {
                // due to division precision loss, should be zero surplus in normal mode
                _cappedColPortion = _cappedColPortion + _collSurplus;
                _collSurplus = 0;
            }
```

### Recovery mode liquidation

The system enters recovery mode as a defensive mode to increase the total collateral ratio of the system, as a result liquidators can further liquidate positions with ICR above the MCR and below the CCR one.

* Lets say we liquidate cdp position with ICR == 120%, the liquidator gets a maximum incentive of 110% and the rest is returned to the cdp owner via surplus.

However in recovery mode we are still free to liquidate cdps below the minimum collateral ratio which can still lead to the rounding error when calculating the surplus. In this case the liquidator will get less incentive collateral while the cdp owner will earn extra surplus which shouldn't be possible when liquidating cdps with ICR < MCR.

```solidity
            if (_collSurplus > 0) {
                collSurplusPool.increaseSurplusCollShares(_borrower, _collSurplus);
                _recoveryState.totalSurplusCollShares =
                    _recoveryState.totalSurplusCollShares +
                    _collSurplus;
            }
```

## Impact Details

l would say the loss here is not significant but rather broken invariant, the system enforces a rule that there should not be any surplus in normal mode which is true as when liquidating cdps with ICR < MCR the whole incentive collateral is supposed to be send to the liquidator. However this invariant doesn't hold in recovery mode, but theoretically it should be the same as the system is allowed to liquidate cdps below the minimum collateral ratio which shouldn't issue any surplus.

## References

<https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/LiquidationLibrary.sol#L336>

## Proof of concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract stormy is eBTCBaseInvariants {

address wallet = address(0xbad455);
uint256 shareEth = 1158379174506084879;
bytes32 underwater;


function setUp() public override {
    super.setUp();

    connectCoreContracts();
    connectLQTYContractsToCore();

}

function testSurplusInRMWhenICRBelowMCR() public {

    // set eth per stETH share
    collateral.setEthPerShare(shareEth);

    // fetch price before open
    uint256 oldprice = priceFeedMock.fetchPrice();

    // open five cdps
    _openTestCDP(wallet, 2e18 + 2e17, ((2e18 * oldprice) / 240e16));
    _openTestCDP(wallet, 2e18 + 2e17, ((2e18 * oldprice) / 240e16));
    _openTestCDP(wallet, 2e18 + 2e17, ((2e18 * oldprice) / 240e16));
    _openTestCDP(wallet, 2e18 + 2e17, ((2e18 * oldprice) / 240e16));
    underwater = _openTestCDP(wallet, 2e18 + 2e17, ((2e18 * oldprice) / 210e16));

    // reduce the price by half to make underwater cdp
    priceFeedMock.setPrice(oldprice / 2);

    // fetch new price after reduce
    uint256 newPrice = priceFeedMock.fetchPrice();

    // ensure the system is in recovery mode
    assert(cdpManager.getSyncedTCR(newPrice) < CCR);

    // liquidate underwater cdp with ICR < MCR
    vm.startPrank(wallet);
    cdpManager.liquidate(underwater);
    vm.stopPrank();

    // make sure the cdp is no longer in the sorted list
    assert(!sortedCdps.contains(underwater));

    // fetch the surplus after the liquidation
    uint256 surplus = collSurplusPool.getSurplusCollShares(wallet);

    // ensure that the surplus is non-zero
    assert(surplus != 0);

    // console log the surplus coll
    console.log("Surplus:", surplus);
    
}
}
```
