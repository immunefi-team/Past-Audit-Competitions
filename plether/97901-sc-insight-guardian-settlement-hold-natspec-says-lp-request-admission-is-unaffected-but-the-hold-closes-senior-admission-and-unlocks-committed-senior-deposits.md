# \#97901 [SC-Insight] Guardian settlement-hold NatSpec says LP request admission is unaffected, but the hold closes Senior admission and unlocks committed Senior deposits

**Submitted on Nov 3rd 2026 at 19:43:14 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97901
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description


## Description

The guardian's emergency LP settlement hold is documented in three places as leaving LP request admission untouched:

- `EmergencyPauseCoordinator.sol:142` (`triggerLpEpochSettlementHold`): "Pauses LP epoch settlement without pausing trading risk **or LP request admission**."
- `interfaces/IHousePoolEmergencyAdmin.sol:12` (`pauseLpEpochSettlement`): "Pauses LP epoch settlement **without pausing LP request admission**."
- `interfaces/IPerpsLPViews.sol:33-34` (`getTrancheQueues`): "The settlement-hold flag is independent from the LP-entry pause and **does not imply request admission is disabled**."

Since commit `d757d0b1` (#123), the hold does change request admission:

1. **Senior requests are refused.** `HousePool._canAcceptTrancheDeposits` returns `false` for Senior while `lpEpochSettlementPaused` is set (`HousePool.sol:480-485`). `TrancheVault.maxRequestDeposit` then returns 0 (`TrancheVault.sol:1319`), and a Senior `requestDeposit` reverts with `TrancheVault__DepositsUnavailable` (`TrancheVault.sol:1596`). Junior requests stay open.
2. **Committed Senior deposits become cancellable.** The `cancelPendingDeposit` gate (`TrancheVault.sol:851-859`) blocks Senior cancellation only while `!POOL.lpEpochSettlementPaused()`. During a hold, an accepted Senior deposit, normally a firm commitment that reverts with `TrancheVault__SeniorDepositCommitted`, can leave escrow.

The implementation's own NatSpec, updated in the same commit, describes this correctly. `HousePool.sol:398`: "Holds new LP epoch clearing **and Senior request admission** without pausing Junior requests or trading." SECURITY.md:879-880 and :492-495 also describe Senior refunds under a hold.

The guardian-facing coordinator and both interfaces were written for the original hold (`153c0f19`, "Add guardian LP settlement hold") and were never updated. So the documentation contradicts itself, and the entrypoint the guardian actually calls is the one that is wrong.

## Conditions under which this insight matters (PoC)

- **Who reads the wrong text:**
  - The guardian and incident runbooks that call `EmergencyPauseCoordinator.triggerLpEpochSettlementHold`.
  - Integrators and front-ends that build on `IHousePoolEmergencyAdmin` / `IPerpsLPViews` (for example, an LP app deciding whether to offer Senior deposits while `getLpStatus` reports a settlement hold).
  - LPs reading the interface docs.
- **When:** whenever a guardian settlement hold is active.
- **Concrete effect:**
  - A client that follows the NatSpec keeps offering Senior deposits during a hold. Every Senior `requestDeposit` then reverts with `TrancheVault__DepositsUnavailable`, so the user pays gas for nothing and sees an unexplained failure.
  - A guardian who triggers the hold expecting it only to stop epoch clearing is not told it also lets Senior LPs pull already-committed capital. That changes Senior capacity and the cash picture during the incident the hold was meant to contain.
- **Evidence:** the Foundry test below uses the production `EmergencyPauseCoordinator` wired as the HousePool pauser, as in deployment, with tranches funded through the normal async LP flow. It shows the guardian's hold closing Senior admission while Junior admission stays open, and a committed Senior deposit that cannot be cancelled before the hold being refunded in full during it.

Place the file at `packages/perps/test/perps/PoC_I1_SettlementHoldRequestAdmission.t.sol` and run:

```
git submodule update --init --recursive
forge test --root packages/perps --match-path test/perps/PoC_I1_SettlementHoldRequestAdmission.t.sol -vv
```


## Recommendation

Bring the three NatSpec blocks in line with `HousePool.sol:398` and SECURITY.md, for example:

- `EmergencyPauseCoordinator.triggerLpEpochSettlementHold` / `IHousePoolEmergencyAdmin.pauseLpEpochSettlement`: "Pauses LP epoch settlement and Senior request admission; already accepted Senior deposits become cancellable. Junior requests and trading continue."
- `IPerpsLPViews.getTrancheQueues`: "During a settlement hold, Senior request admission is closed and accepted Senior deposits may be cancelled; Junior admission is unaffected."

## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Insight PoC: the guardian's LP settlement hold DOES stop Senior request admission (and unlocks committed Senior
// deposits), but EmergencyPauseCoordinator.triggerLpEpochSettlementHold, IHousePoolEmergencyAdmin.pauseLpEpochSettlement
// and IPerpsLPViews.getTrancheQueues all document that it does not pause / does not imply disabling LP request
// admission. HousePool's own NatSpec (HousePool.sol:398) describes the real behaviour, so the documentation contradicts
// itself and the guardian-facing entrypoint is the one that is wrong.
//
// Uses the repo's BasePerpTest stack plus the production EmergencyPauseCoordinator wired as the HousePool pauser, as in
// deployment. Tranches are funded through the normal async LP flow. Only Pyth and USDG are mocks.

import {BasePerpTest} from "./BasePerpTest.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

contract PoC_I1_SettlementHoldRequestAdmission is BasePerpTest {

    address internal constant GUARDIAN = address(0x6A2D);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant JUNIOR_LP = address(0x1A1A);
    address internal constant SENIOR_LP = address(0x5E17);
    address internal constant NEW_SENIOR = address(0x5E18);
    address internal constant NEW_JUNIOR = address(0x1A1B);

    EmergencyPauseCoordinator internal coordinator;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // Production wiring: the coordinator is the HousePool's dedicated pauser and the guardian triggers it.
        coordinator = new EmergencyPauseCoordinator(address(routerAdmin), address(pool), address(this));
        coordinator.setGuardian(GUARDIAN);
        pool.setPauser(address(coordinator));
        _seedTranche(juniorVault, JUNIOR_LP, 2_000_000e6);
        _seedTranche(seniorVault, SENIOR_LP, 6_000_000e6);
    }

    function _refreshMark() internal {
        bytes[] memory updateData = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.updateMarkPrice(updateData);
    }

    function _seedTranche(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        _refreshMark();
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId) + 5);
        _refreshMark();
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        vm.prank(lp);
        vault.claimDeposit(requestId, assets, lp, lp);
    }

    function _fundAndApprove(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        settlementToken.mint(lp, assets);
        vm.prank(lp);
        settlementToken.approve(address(vault), assets);
    }

    /// @notice NatSpec: the hold works "without pausing ... LP request admission". Code: it closes Senior admission.
    function test_I1_guardianHoldClosesSeniorRequestAdmission() public {
        _refreshMark();
        assertTrue(pool.canAcceptTrancheDeposits(true), "Senior admission open before the hold");
        assertTrue(pool.canAcceptTrancheDeposits(false), "Junior admission open before the hold");

        vm.prank(GUARDIAN);
        coordinator.triggerLpEpochSettlementHold(bytes32(0), bytes32(0));
        assertTrue(pool.lpEpochSettlementPaused(), "hold active");

        // Senior request admission is closed by the hold ...
        assertFalse(pool.canAcceptTrancheDeposits(true), "hold closes Senior admission");
        assertEq(seniorVault.maxRequestDeposit(NEW_SENIOR), 0, "Senior maxRequestDeposit is zero");
        _fundAndApprove(seniorVault, NEW_SENIOR, 10_000e6);
        vm.prank(NEW_SENIOR);
        vm.expectRevert(TrancheVault.TrancheVault__DepositsUnavailable.selector);
        seniorVault.requestDeposit(10_000e6, NEW_SENIOR, NEW_SENIOR);

        // ... while Junior admission stays open, matching HousePool.sol:398 but not the coordinator/interface NatSpec.
        assertTrue(pool.canAcceptTrancheDeposits(false), "Junior admission stays open");
        _fundAndApprove(juniorVault, NEW_JUNIOR, 10_000e6);
        vm.prank(NEW_JUNIOR);
        juniorVault.requestDeposit(10_000e6, NEW_JUNIOR, NEW_JUNIOR);
    }

    /// @notice A second effect the guardian-facing NatSpec omits: committed Senior deposits become cancellable.
    function test_I1_guardianHoldUnlocksCommittedSeniorDeposits() public {
        _refreshMark();
        _fundAndApprove(seniorVault, NEW_SENIOR, 10_000e6);
        vm.prank(NEW_SENIOR);
        uint256 requestId = seniorVault.requestDeposit(10_000e6, NEW_SENIOR, NEW_SENIOR);

        // Without a hold an accepted Senior deposit is a firm commitment.
        vm.prank(NEW_SENIOR);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(requestId);

        vm.prank(GUARDIAN);
        coordinator.triggerLpEpochSettlementHold(bytes32(0), bytes32(0));

        uint256 walletBefore = settlementToken.balanceOf(NEW_SENIOR);
        vm.prank(NEW_SENIOR);
        uint256 refunded = seniorVault.cancelPendingDeposit(requestId);
        assertEq(refunded, 10_000e6, "the hold lets the committed Senior deposit leave escrow");
        assertEq(settlementToken.balanceOf(NEW_SENIOR) - walletBefore, 10_000e6);
    }

}
```
