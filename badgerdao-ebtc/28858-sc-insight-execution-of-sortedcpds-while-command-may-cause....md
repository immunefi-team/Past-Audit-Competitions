# 28858 - \[SC - Insight] Execution of SortedCpds while command may cause...

Submitted on Feb 29th 2024 at 01:10:58 UTC by @cryptoticky for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28858

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/SortedCdps.sol>

Impacts:

* Unbounded gas consumption

## Description

## Brief/Intro

Execution of SortedCpd's while command may cause excessive gas consumption.

## Vulnerability Details

The codes below can result in significant gas costs.

CdpManager.sol:373-376 lines

```solidity
while (currentBorrower != address(0) && getSyncedICR(_cId, totals.price) < MCR) {
    _cId = sortedCdps.getPrev(_cId);
    currentBorrower = sortedCdps.getOwnerAddress(_cId);
}
```

HintHelpers.sol:68-74 lines

```solidity
while (
    vars.currentCdpUser != address(0) &&
    cdpManager.getSyncedICR(vars.currentCdpId, _price) < MCR
) {
    vars.currentCdpId = sortedCdps.getPrev(vars.currentCdpId);
    vars.currentCdpUser = sortedCdps.getOwnerAddress(vars.currentCdpId);
}
```

If there are a significant number of CDPs with ICRs smaller than MCRs, the user must pay a significant gas cost and out of gas exception can occur, resulting in gas loss. In the worst-case scenario, the gas cost may exceed the block gas limit and the protocol will not be able to operate normally. However, the latter is theoretically possible, but will not happen in reality.

In this report, the former is explained and solutions are presented.

Let's look at the cases (no CDP and 100 CDPs) : ICR < MCR

You can create PoC\_CDPManager.redemptions.gaslimit.t.sol file in foundry\_test folder.

And run this in terminal.

`forge test -vvvv --match-contract PoC_CDPManagerRedemptionsGasLimitTest`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "../contracts/Dependencies/EbtcMath.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract PoC_CDPManagerRedemptionsGasLimitTest is eBTCBaseInvariants {
    address user;


    function setUp() public override {
        super.setUp();
        connectCoreContracts();
        connectLQTYContractsToCore();
        vm.warp(3 weeks);
    }

    function test_PoC1SingleRedemptionGasFee() public {
        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();


        bytes32 cdpId0;
        bytes32 cdpId1;
        bytes32 cdpId2;

        console.log("\n");
        console.log("------- Redemption when there is no CDP (ICR < MCR) -------");

        // --- open cdps ---
        (, cdpId1) = _singleCdpSetupWithICR(user, 130e16);
        (, cdpId2) = _singleCdpSetupWithICR(user, 140e16);

        console.log("- Redeem CDP1 fully");
        bytes32 dummyId = sortedCdps.dummyId();
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId1);
        (, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.startPrank(user);
        cdpManager.redeemCollateral(
            _redeemedDebt,
            dummyId,
            cdpId1,
            cdpId1,
            partialRedemptionHintNICR,
            0,
            1e18
        );
        vm.stopPrank();

        console.log("Please check gas amounts in the test report!!!");

        console.log("The report must show these result!");
        console.log("--- Result ---");
        console.log("[283703] AccruableCdpManager::redeemCollateral");
    }

    function test_PoC2SingleRedemptionGasFee() public {
        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();


        console.log("\n");
        console.log("------- Redemption when there are 100 CDPs (ICR < MCR) -------");

        // --- open cdps ---
        bytes32 cdpId1;
        for (uint256 i; i < 1000; i++) {
            (, cdpId1) = _singleCdpSetupWithICR(user, 130e16);
        }

        uint256 price = priceFeedMock.getPrice();
        price = price * 10 / 12;
        priceFeedMock.setPrice(price);

        (, bytes32 cdpId2) = _singleCdpSetupWithICR(user, 130e16);
        for (uint256 j; j < 10; j++) {
            _singleCdpSetupWithICR(user, 400e16);
        }
        

        uint256 icr1 = cdpManager.getSyncedICR(cdpId1, price);
        uint256 icr2 = cdpManager.getSyncedICR(cdpId2, price);

        console.log("CDP1's ICR: ", icr1);
        console.log("CDP2's ICR: ", icr2);
        console.log("There are 100 CDPs (ICR < MCR)");

        console.log("- Redeem CDP2 fully");
        bytes32 dummyId = sortedCdps.dummyId();
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId2);
        (, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.startPrank(user);
        cdpManager.redeemCollateral(
            _redeemedDebt,
            dummyId,
            cdpId2,
            cdpId2,
            partialRedemptionHintNICR,
            0,
            1e18
        );
        vm.stopPrank();

        console.log("Please check gas amounts in the test report!!!");

        console.log("The report must show these result!");
        console.log("--- Result ---");
        console.log("[1066824] AccruableCdpManager::redeemCollateral");

        console.log("[556] SortedCdps::getPrev");
        console.log("[332] SortedCdps::getOwnerAddress");
        console.log("[957] CollateralTokenTester::getPooledEthByShares");
        console.log("[957] CollateralTokenTester::getPooledEthByShares");

        console.log("!!!This is causing 3.7 (1066824 / 283703) times the cost of gas!!!");
    }


    function _singleCdpSetupWithICR(address _usr, uint256 _icr) internal returns (address, bytes32) {
        uint256 _price = priceFeedMock.fetchPrice();
        uint256 _coll = cdpManager.MIN_NET_STETH_BALANCE() * 2;
        uint256 _debt = (_coll * _price) / _icr;
        bytes32 _cdpId = _openTestCDP(_usr, _coll + cdpManager.LIQUIDATOR_REWARD(), _debt);
        uint256 _cdpICR = cdpManager.getCachedICR(_cdpId, _price);
        return (_usr, _cdpId);
    }

    function _performRedemption(
        address _redeemer,
        uint256 _redeemedDebt,
        bytes32 _upperPartialRedemptionHint,
        bytes32 _lowerPartialRedemptionHint
    ) internal {
        (bytes32 firstRedemptionHint, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.prank(_redeemer);
        cdpManager.redeemCollateral(
            _redeemedDebt,
            firstRedemptionHint,
            _upperPartialRedemptionHint,
            _lowerPartialRedemptionHint,
            partialRedemptionHintNICR,
            0,
            1e18
        );
    }
}

```

It can be seen that when there are 100 CDPs with ICR < MCR, the gas cost is about 3.7 times higher than when there is no. If 1000 CDPs were present, it would take about 37 times more gas.

## Impact Details

#### 1. The user has to pay a considerable fee for gas.

#### 2. It is easy to cause gas loss due to out of gas exception.

As you can see, the gas amount is 1066824 with 100 CDPs (ICR < MCR).

This transaction is created at Feb-28-2024 06:25:23 PM +UTC.

<https://etherscan.io/tx/0x86cbd747180c6c7ce935fbdf853b87e0addfa3046633071e16cb9ed941e4e21c>

The gas price is 139.797022618 Gwei.

1066824 \* 139.797022618 Gwei = 149,138,818,857,425,232

ether price is $3,350 so gas cost = $3,350 \* 0.149 = $499.15

This increases arithmetic with more CDP (ICR < MCR).

If there are 1000 CDPs, the gas cost is about $5,000. :)

## Recommend

### Solution 1: `Binary search engine`

I recommend you use `binary search` engine to search the FirstCDP (ICR >= MCR) in the code.

Fortunately, Openzeppelin provides useful library for our problem.

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.6.0/contracts/utils/Arrays.sol>

```solidity
library Arrays {
    /**
     * @dev Searches a sorted `array` and returns the first index that contains
     * a value greater or equal to `element`. If no such index exists (i.e. all
     * values in the array are strictly less than `element`), the array length is
     * returned. Time complexity O(log n).
     *
     * `array` is expected to be sorted in ascending order, and to contain no
     * repeated elements.
     */
    function findUpperBound(uint256[] storage array, uint256 element) internal view returns (uint256) {
        if (array.length == 0) {
            return 0;
        }

        uint256 low = 0;
        uint256 high = array.length;

        while (low < high) {
            uint256 mid = Math.average(low, high);

            // Note that mid will always be strictly less than high (i.e. it will be a valid array index)
            // because Math.average rounds down (it does integer division with truncation).
            if (array[mid] > element) {
                high = mid;
            } else {
                low = mid + 1;
            }
        }

        // At this point `low` is the exclusive upper bound. We will return the inclusive upper bound.
        if (low > 0 && array[low - 1] == element) {
            return low - 1;
        } else {
            return low;
        }
    }
}
```

The number of calculations is log2(n).

n is number of items in array.

#### Solution 2

Every time a liquidation is made, we can store last CDP (ICR < MCR) position.

When price is changed, the IRC and last CDP would be changed.

But based on the stored last CDP, the search can be carried out back and forth.

This does not require many modifications to SortedCDPs contact than Solution 1, and will also drastically reduce the number of computations. It may be more effective to update this variable whenever there is a change in CDP due to various operations.

## Proof of Concept

Let's look at the cases (no CDP and 100 CDPs) : ICR < MCR

You can create PoC\_CDPManager.redemptions.gaslimit.t.sol file in foundry\_test folder.

And run this in terminal.

`forge test -vvvv --match-contract PoC_CDPManagerRedemptionsGasLimitTest`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "../contracts/Dependencies/EbtcMath.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract PoC_CDPManagerRedemptionsGasLimitTest is eBTCBaseInvariants {
    address user;


    function setUp() public override {
        super.setUp();
        connectCoreContracts();
        connectLQTYContractsToCore();
        vm.warp(3 weeks);
    }

    function test_PoC1SingleRedemptionGasFee() public {
        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();


        bytes32 cdpId0;
        bytes32 cdpId1;
        bytes32 cdpId2;

        console.log("\n");
        console.log("------- Redemption when there is no CDP (ICR < MCR) -------");

        // --- open cdps ---
        (, cdpId1) = _singleCdpSetupWithICR(user, 130e16);
        (, cdpId2) = _singleCdpSetupWithICR(user, 140e16);

        console.log("- Redeem CDP1 fully");
        bytes32 dummyId = sortedCdps.dummyId();
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId1);
        (, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.startPrank(user);
        cdpManager.redeemCollateral(
            _redeemedDebt,
            dummyId,
            cdpId1,
            cdpId1,
            partialRedemptionHintNICR,
            0,
            1e18
        );
        vm.stopPrank();

        console.log("Please check gas amounts in the test report!!!");

        console.log("The report must show these result!");
        console.log("--- Result ---");
        console.log("[283703] AccruableCdpManager::redeemCollateral");
    }

    function test_PoC2SingleRedemptionGasFee() public {
        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();


        console.log("\n");
        console.log("------- Redemption when there are 100 CDPs (ICR < MCR) -------");

        // --- open cdps ---
        bytes32 cdpId1;
        for (uint256 i; i < 1000; i++) {
            (, cdpId1) = _singleCdpSetupWithICR(user, 130e16);
        }

        uint256 price = priceFeedMock.getPrice();
        price = price * 10 / 12;
        priceFeedMock.setPrice(price);

        (, bytes32 cdpId2) = _singleCdpSetupWithICR(user, 130e16);
        for (uint256 j; j < 10; j++) {
            _singleCdpSetupWithICR(user, 400e16);
        }
        

        uint256 icr1 = cdpManager.getSyncedICR(cdpId1, price);
        uint256 icr2 = cdpManager.getSyncedICR(cdpId2, price);

        console.log("CDP1's ICR: ", icr1);
        console.log("CDP2's ICR: ", icr2);
        console.log("There are 100 CDPs (ICR < MCR)");

        console.log("- Redeem CDP2 fully");
        bytes32 dummyId = sortedCdps.dummyId();
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId2);
        (, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.startPrank(user);
        cdpManager.redeemCollateral(
            _redeemedDebt,
            dummyId,
            cdpId2,
            cdpId2,
            partialRedemptionHintNICR,
            0,
            1e18
        );
        vm.stopPrank();

        console.log("Please check gas amounts in the test report!!!");

        console.log("The report must show these result!");
        console.log("--- Result ---");
        console.log("[1066824] AccruableCdpManager::redeemCollateral");

        console.log("[556] SortedCdps::getPrev");
        console.log("[332] SortedCdps::getOwnerAddress");
        console.log("[957] CollateralTokenTester::getPooledEthByShares");
        console.log("[957] CollateralTokenTester::getPooledEthByShares");

        console.log("!!!This is causing 3.7 (1066824 / 283703) times the cost of gas!!!");
    }


    function _singleCdpSetupWithICR(address _usr, uint256 _icr) internal returns (address, bytes32) {
        uint256 _price = priceFeedMock.fetchPrice();
        uint256 _coll = cdpManager.MIN_NET_STETH_BALANCE() * 2;
        uint256 _debt = (_coll * _price) / _icr;
        bytes32 _cdpId = _openTestCDP(_usr, _coll + cdpManager.LIQUIDATOR_REWARD(), _debt);
        uint256 _cdpICR = cdpManager.getCachedICR(_cdpId, _price);
        return (_usr, _cdpId);
    }

    function _performRedemption(
        address _redeemer,
        uint256 _redeemedDebt,
        bytes32 _upperPartialRedemptionHint,
        bytes32 _lowerPartialRedemptionHint
    ) internal {
        (bytes32 firstRedemptionHint, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.prank(_redeemer);
        cdpManager.redeemCollateral(
            _redeemedDebt,
            firstRedemptionHint,
            _upperPartialRedemptionHint,
            _lowerPartialRedemptionHint,
            partialRedemptionHintNICR,
            0,
            1e18
        );
    }
}

```

It can be seen that when there are 100 CDPs with ICR < MCR, the gas cost is about 3.7 times higher than when there is no. If 1000 CDPs were present, it would take about 37 times more gas.

## Impact Details

#### 1. The user has to pay a considerable fee for gas.

#### 2. It is easy to cause gas loss due to out of gas exception.

As you can see, the gas amount is 1066824 with 100 CDPs (ICR < MCR).

This transaction is created at Feb-28-2024 06:25:23 PM +UTC.

<https://etherscan.io/tx/0x86cbd747180c6c7ce935fbdf853b87e0addfa3046633071e16cb9ed941e4e21c>

The gas price is 139.797022618 Gwei.

1066824 \* 139.797022618 Gwei = 149,138,818,857,425,232

ether price is $3,350 so gas cost = $3,350 \* 0.149 = $499.15

This increases arithmetic with more CDP (ICR < MCR).

If there are 1000 CDPs, the gas cost is about $5,000.
