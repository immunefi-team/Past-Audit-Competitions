# 28823 - \[SC - Insight] Lido slashing can negatively affect the whole l...

Submitted on Feb 28th 2024 at 01:49:05 UTC by @Stormy for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28823

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/LiquidationLibrary.sol>

Impacts:

* Temporary freezing of funds for at least 15 minutes

## Description

## Brief/Intro

Lido slashing can negatively affect the underlying value of all cdp liquidator rewards and all cdp collateral shares making liquidations less profitable or non-profitable especially for small cdp positions. The liquidations are the core concept for the system to escape unwanted state (RM), any problems there can negatively affect the whole protocol and its users.

If the system enters recovery mode after slashing, the only way out would be for the protocol team to liquidate all the non-profitable small cdp positions and take the loss or increasing the TCR via opening or repaying debt otherwise withdrawing any funds from the system will not be allowed until escaping RM.

## Vulnerability Details

Prior to opening a cdp in the protocol, the system allocates a fixed amount of liquidator reward with value of 0.2 stETH. This reward is given to the liquidator which liquidates the position in a case the overall ICR of the cdp drops below a certain point based on the current mode of the system. The primary goal of this amount of collateral is to compensate the liquidator from the gas he is paying to liquidate the cdp position but it can still be counted for further profit on small cdps.

```solidity
        uint256 _liquidatorRewardShares = collateral.getSharesByPooledEth(LIQUIDATOR_REWARD);
```

On short explanation eBTC uses a rebasing token called stETH for its core collateral and tracks the shares of the token to correctly account the rebasing logic. So one share of stETH can hold bigger value of eth in the future than it holds now.

Currently the system tracks the value of 1 stETH share and syncs the global accounting on every cdp operation if needed. Having a greater value of 1 stETH share than before indicates that a positive rebase happened and that the system needs to sync the global accounting and take its split fee. However there could be cases when the value of 1 stETH share reduces this indicates that a slashing happened in Lido and as a result the shares reduced their underlying value of eth in this case the eBTC syncs the indexes without taking any split fee.

```solidity
    function _syncGlobalAccounting() internal {
        (uint256 _oldIndex, uint256 _newIndex) = _readStEthIndex();
        _syncStEthIndex(_oldIndex, _newIndex);
        if (_newIndex > _oldIndex && totalStakes > 0) {
            (
                uint256 _feeTaken,
                uint256 _newFeePerUnit,
                uint256 _perUnitError
            ) = _calcSyncedGlobalAccounting(_newIndex, _oldIndex);
            _takeSplitAndUpdateFeePerUnit(_feeTaken, _newFeePerUnit, _perUnitError);
            _updateSystemSnapshotsExcludeCollRemainder(0);
        }
    }
```

But even tho the system correctly accounts the cases when a negative rebase occurs (slashing), there would be still some internal damage done on the overall underlying value of all cdp collaterals and liquidator rewards.

l would say the bigger problem here with slashing lays in all cdp positions with minimum collateral of 2 stETH, so take as example that Lido slash occurs which reduces the value of the stETH share by 30%.

* Depending on the TCR after this percent of slashing the system may or may not enter recovery mode.
* Duo to the slash all cdps underlying value of collateral will also be reduced by 30%.
* And the underlying value of all liquidator rewards will also be reduced by 30%.

With 30% slashing, all 2 stETH cdps with collateral ratio below 130% will become underwater with bad debt immedately, not only that but their underlying value of liquidator reward and cdp collateral shares will be reduced by 30% as well.

Liquidators receive 3% discount on the debt to repay when liquidating a cdp with ICR <= LICR, but in our case considering the 3% of the reduced by 30% collateral value of 2 stETH position it will be close to nothing. In this situation for the liquidator to make profit he will be more dependant on the liquidator reward which will also be reduced after the slash.

```solidity
        } else {
            // for full liquidation, there would be some bad debt to redistribute
            _incentiveColl = collateral.getPooledEthByShares(_totalColToSend);

            // Since it's full and there's bad debt we use spot conversion to
            // Determine the amount of debt that willl be repaid after adding the LICR discount
            // Basically this is buying underwater Coll
            // By repaying debt at 3% discount
            // Can there be a rounding error where the _debtToRepay > debtToBurn?
            uint256 _debtToRepay = (_incentiveColl * _price) / LICR;
```

Liquidations is the main logic with which the system escapes RM, if one severe slash leads close to non-profitable liquidations especially for small cdp positions of 2 stETH. There would not be any incentive for liquidators to liquidate this underwater positions and help the system to step back on its feet.

## Impact Details

While the issue with the slashing is not a big problem for bigger cdp positions, it can negatively affect the smaller cdp positions with minimum collateral as the liquidators are mainly dependant on the liquidator reward in order to make profit.

It will be hard for the system to survive more severe slashing, as once the system hits RM and there are dozens of minimum underwater cdp positions with less than 2 stETH collateral value (based on the percent slashing), the only way out might be to either for the protocol team to manually liquidate them and take the loss or increasing the TCR via opening or repaying debt, otherwise unless the system escapes recovery mode there won't be a way to withdraw collateral from the cdp positions.

* Just to clarify as a fact that slashing can negatively affect the underlying value of all cdp liquidator reward shares, take as example that a 35% slash happens, with 3.5% yearly APR the liquidator shares will need 10 years of rebasing to gain their original value back before the slashing.

```solidity
// BorrowerOperations -> when closing cdp

        _requireNotInRecoveryMode(_getCachedTCR(price));
```

```solidity
// BorrowerOperations -> when withdrawing coll with adjusting

        if (_isRecoveryMode) {
            _requireNoStEthBalanceDecrease(_stEthBalanceDecrease);
```

## References

<https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/BorrowerOperations.sol#L470> <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/LiquidationLibrary.sol#L592-#L601>

## Proof of concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";


contract stormy is eBTCBaseInvariants {

    address wallet = address(0xbad455);
    uint256 ethPerShare;

    function setUp() public override {
        super.setUp();

        connectCoreContracts();
        connectLQTYContractsToCore();

        eBTCToken.approve(address(cdpManager), type(uint256).max);
    }

    function testSlashingIssue() public {

        // set eth per share as stETH
        collateral.setEthPerShare(1158489314227548015);

        // sync global accounting
        vm.startPrank(address(borrowerOperations));
        cdpManager.syncGlobalAccounting();
        vm.stopPrank();

        // save the eth per share
        ethPerShare = cdpManager.stEthIndex();

        // fetch price
        uint256 price = priceFeedMock.fetchPrice();

        // open 100 cdps with ICR starting from 225% to 125%
        uint256 count = 100;
        uint256 ratio = 225e16;
        while(count > 0){
            _openTestCDP(wallet, 10e18 + 2e17, ((10e18 * price) / ratio));
            count--;
            ratio -= 1e16;
        }

        // open 50 small cdps with min coll of 2 stETH 
        count += 50;
        bytes32 lastCdp;
        while(count > 0){
            lastCdp = _openTestCDP(wallet, 2e18 + 2e17, ((2e18 * price) / 130e16));
            count--;
        }

        // ensure the TCR is above the CCR
        assert(cdpManager.getSyncedTCR(price) > CCR);

        // fetch the 2 stETH cdp's coll and liquidator reward
        uint256 collCdp = cdpManager.getSyncedCdpCollShares(lastCdp);
        uint256 liqReward = cdpManager.getCdpLiquidatorRewardShares(lastCdp);

        // get the underlying value of eth from the coll and liquidator reward shares
        uint256 beforeUnderlyingValueColl = collateral.getEthPerShare() * collCdp;
        uint256 beforeUnderlyingValueLiqReward = collateral.getEthPerShare() * liqReward;

        // console log the underlying value of coll and liquidator reward before the slash
        console.log("===============================Before");
        console.log("BeforeSlashCdpValueColl          :",beforeUnderlyingValueColl);
        console.log("BeforeSlashCdpValueLiqReward     :",beforeUnderlyingValueLiqReward);

        // slash by 33.3% the underlying value of the stETH share
        collateral.setEthPerShare(ethPerShare - (ethPerShare / 3)); // 33.3% reduce 

        // get the underlying value of eth from the coll and liquidator reward share after the slash
        uint256 afterUnderlyingValueColl = collateral.getEthPerShare() * collCdp;
        uint256 afterUnderlyingValueLiqReward = collateral.getEthPerShare() * liqReward;

        // console log the underlying value of coll and liquidator reward after the slash
        console.log("===============================After");
        console.log("AfterSlashCdpValueColl           :",afterUnderlyingValueColl);
        console.log("AfterSlashCdpValueLiqReward      :",afterUnderlyingValueLiqReward);

        // ensure the TCR drops below the CCR after the slashing
        assert(cdpManager.getSyncedTCR(price) < CCR);

        // calculate the three percent incentive in underlying value of eth
        uint256 LICR = 103e16;
        uint256 debtToRepay = ((afterUnderlyingValueColl * price) / LICR);
        uint256 threePercentCollIncentive = afterUnderlyingValueColl - (debtToRepay * DECIMAL_PRECISION) / price;

        // console log the underlying value of the liq reward and three percent incentive
        console.log("====================================");
        console.log("LiqRewardUnderlyingValue         :", afterUnderlyingValueLiqReward);
        console.log("ThreePercentIncenvtiveColl       :", threePercentCollIncentive);    

        // calculate total proft of underlying value of the liquidation
        uint256 profitFromLiq = threePercentCollIncentive + afterUnderlyingValueLiqReward;

        // console log the total profit after the liquidation
        console.log("====================================");
        console.log("TotalProfitFromLiq               :", profitFromLiq);
    }
}
```
