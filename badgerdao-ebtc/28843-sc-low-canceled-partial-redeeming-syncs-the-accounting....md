# 28843 - \[SC - Low] Canceled partial redeeming syncs the accounting...

Submitted on Feb 28th 2024 at 16:02:19 UTC by @Stormy for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28843

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/CdpManager.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

l would say this bug breaks an important invariant as the stake of every cdp is supposed to be updated on every cdp operation, duo to the fact that the stake ratio always changes with every split fee. This issue will allow the user to pay less or simply wrong collateral fee onwards as the split fee owned by the cdp is calculated based on its stake.

## Vulnerability Details

The total stake variable is used for the determination of the correct amount of collateral fee that needs to be payed per unit staked and every cdp has an amount of stake which correspondents to the amount of collateral the position has. As the system takes its split fee from the total collateral shares, this stake ratio changes as a result when doing any cdp operation the system correctly syncs and adjusts the collateral shares of the cdp and updates its stake based on the new stake ratio.

By going over the protocol logic, we can notice that on every cdp operation the system updates the cdp's stake:

* When adjusting a cdp the system calculates and updates the cdp's stake based on the latest stake ratio and the amount of coll shares the cdp is left with after the adjusting.
* When partial liquidating a cdp the system calculates and updates the cdp's stake based on the latest stake ratio and the amount of coll shares the cdp is left with after the partial liquidation.
* When successfully partially redeeming from a cdp the system calculates and updates the cdp's stake based on the latest stake ratio and the amount of coll shares the cdp is left with after the partial redemption.

The problem we are facing occurs when partial redeeming, before every redemption happens the system successfully syncs the accounting of the particular cdp to accordingly adjust its collateral shares for the redemption. Currently there could be two outcomes when partial redeeming:

* The partial redemption is successful as a result the system updates the cdp's accounting to correspondent the new values the position has.
* The partial redemption is canceled, this can happen when the user provides a wrong NICR hint or if the partial redeeming either drops the cdp collateral below the minimum balance of 2 stETH or if its debt drops below the minimum change of 1000 wei.

In a case when partial redeeming is canceled the system doesn't revert but returns, in this case the particular cdp will keep its synced stats but the system misses to update its stake to correspondent the synced collateral shares and the latest stake ratio. This can be problematic considering that the split fee the cdp owns will be calculated based on the wrong stake when the next positive rebase happens.

```solidity
        } else {
            // Debt remains, reinsert Cdp
            uint256 newNICR = EbtcMath._computeNominalCR(newColl, newDebt);

            /*
             * If the provided hint is out of date, we bail since trying to reinsert without a good hint will almost
             * certainly result in running out of gas.
             *
             * If the resultant net coll of the partial is less than the minimum, we bail.
             */
            if (
                newNICR != _redeemColFromCdp.partialRedemptionHintNICR ||
                collateral.getPooledEthByShares(newColl) < MIN_NET_STETH_BALANCE ||
                newDebt < MIN_CHANGE
            ) {
                singleRedemption.cancelledPartial = true;
                return singleRedemption;
            }

            singleRedemption.newPartialNICR = newNICR;

            Cdps[_redeemColFromCdp.cdpId].debt = newDebt;
            Cdps[_redeemColFromCdp.cdpId].coll = newColl;
            _updateStakeAndTotalStakes(_redeemColFromCdp.cdpId);

            emit CdpUpdated(
                _redeemColFromCdp.cdpId,
                ISortedCdps(sortedCdps).getOwnerAddress(_redeemColFromCdp.cdpId),
                msg.sender,
                _oldDebtAndColl.debt,
                _oldDebtAndColl.collShares,
                newDebt,
                newColl,
                Cdps[_redeemColFromCdp.cdpId].stake,
                CdpOperation.redeemCollateral
            );
        }
```

## Impact Details

The stake functionality is crucial for the protocol as both the split fee and bad debt is calculated based on it. So it is mandatory to keep this value of stake as accurate as possible. Not updating a cdp stake after syncing the accounting may lead to the cdp paying a wrong amount of split fee or bad debt next time. As mentioned in my brief/info section this is more like an invariant that needs to hold as on every cdp operation the system updates the cdp's stake.

## References

<https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/CdpManager.sol#L190-L197>

## Proof of concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract stormy is eBTCBaseInvariants {

address wallet = address(0xbad455);
uint256 yearlyIncrease = 36000000000000000;
uint256 dailyIncrease = yearlyIncrease / 365;
uint256 ethPerShare = 1e18;
bytes32 newCdp;
bytes32 partialCdp;
uint256 price;
uint256 grossColl;
uint256 debtMCR;
uint256 debtColl;
bytes32 last;

function setUp() public override {
    super.setUp();

    connectCoreContracts();
    connectLQTYContractsToCore();
}

function testSortingForPartialRedemption() public {

    // fetch the price
    price = priceFeedMock.fetchPrice();

    // calculate the net coll + liquidator reward
    grossColl = 10e18 + cdpManager.LIQUIDATOR_REWARD();

    // calculate the debt allowed with collateral ratio of 110%
    debtMCR = _utils.calculateBorrowAmount(10e18, price, MINIMAL_COLLATERAL_RATIO);

    // calculate the debt allowed with collateral ratio of 125%
    debtColl = _utils.calculateBorrowAmount(10e18, price, COLLATERAL_RATIO);

    // open 5 cdps in the system
    _openTestCDP(wallet, grossColl, debtColl - 20000);
    _openTestCDP(wallet, grossColl, debtColl - 10000);
    _openTestCDP(wallet, grossColl, debtColl - 7500);
    partialCdp = _openTestCDP(wallet, grossColl, debtColl);
    _openTestCDP(wallet, grossColl, debtMCR);

    vm.startPrank(wallet);

    // approve ebtc token to cdp manager
    eBTCToken.approve(address(cdpManager), eBTCToken.balanceOf(wallet));

    // increase the eth per share with one day split fee
    collateral.setEthPerShare(ethPerShare + dailyIncrease);

    // get the last Cdp in the system
    last = sortedCdps.getLast();

    // record the old coll and stake of the partial redeemed cdp
    uint256 oldCollBefore = cdpManager.getCdpCollShares(partialCdp);
    uint256 oldStakeBefore = cdpManager.getCdpStake(partialCdp);

    // sync the debt twap spot value
    _syncSystemDebtTwapToSpotValue();

    // fully redeem the last cdp and try to partial redeem but cancel with wrong NICR
    cdpManager.redeemCollateral(
        debtMCR + (debtColl / 2),
        last,
        bytes32(0),
        bytes32(0),
        0,
        0,
        1e18
    );

    vm.stopPrank();

    // record the new coll and stake after the canceled partial redeeming
    uint256 newCollAfter = cdpManager.getCdpCollShares(partialCdp);
    uint256 newStakeAfter = cdpManager.getCdpStake(partialCdp);

    // ensure that the last cdp was fully redeemed
    assert(!sortedCdps.contains(last));

    // console log the old and new stats of the canceled cdp
    console.log("================Before");
    console.log("CollBefore            :", oldCollBefore);
    console.log("StakeBefore           :", oldStakeBefore);
    console.log("================After"); 
    console.log("CollAfter             :", newCollAfter);
    console.log("StakeAfter            :", newStakeAfter);

    // fetch the stake and coll snapshots
    uint256 stakeSnapshot = cdpManager.totalStakesSnapshot();
    uint256 collSnapshot = cdpManager.totalCollateralSnapshot();

    // calculate the correct stake based on the cdp new coll and latest stake ratio
    uint256 correctStake = (newCollAfter * stakeSnapshot) / collSnapshot;

    // console log the correct stake
    console.log("=====================");     
    console.log("CorrectStake          :", correctStake);

    // The diference is based on 5 cdps in the system and only one daily increase.
}
}
```
