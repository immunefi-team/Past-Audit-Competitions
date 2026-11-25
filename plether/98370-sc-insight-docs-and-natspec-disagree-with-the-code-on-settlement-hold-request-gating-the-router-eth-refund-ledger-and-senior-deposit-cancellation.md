# \#98370 [SC-Insight] Docs and NatSpec disagree with the code on settlement-hold request gating, the router ETH-refund ledger, and Senior deposit cancellation

**Submitted on Nov 7th 2026 at 12:43:36 UTC by @janneh2000 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98370
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Asset**

https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src

**Impact**

Documentation Improvements Insight

**Description**

While checking the perps docs against the code at a82d4fef, I found three places where the written behaviour differs from what the contracts do. None of them moves value; they mislead integrators, keepers and monitoring that rely on the docs.

1. The NatSpec says the LP settlement hold leaves request admission open; the code closes Senior deposit admission. SECURITY.md's wording is ambiguous.

NatSpec: triggerLpEpochSettlementHold "Pauses LP epoch settlement without pausing trading risk or LP request admission" (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/EmergencyPauseCoordinator.sol#L142), and PerpsPublicLens says a hold defers activation and funding "without disabling request admission" (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PerpsPublicLens.sol#L295-L296).
Code: _canAcceptTrancheDeposits returns false for Senior while lpEpochSettlementPaused (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L483-L485), so maxRequestDeposit is 0 and Senior requestDeposit reverts with DepositsUnavailable during a hold (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1318, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1594-L1597). Senior requestRedeem and maxRequestRedeem have no hold check, so Senior redemption requests stay admissible (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L630, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1328-L1335).
SECURITY.md says the hold "stops new Senior requests and reservations" (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L496-L498) and "blocks new Senior requests", listing only Junior deposit and redemption requests as live (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1154-L1156). Read literally, that includes Senior redemption requests, which the code does not block.
Suggested fix: state in all three places that the hold blocks new Senior deposit requests and reservations, keeps Senior and Junior redemption requests and Junior deposit requests admissible, and defers their funding and activation until release.

2. Router entrypoint NatSpec says the oracle refunds excess ETH and records the deferred claim; the router does both.

NatSpec: IOrderRouter.updateMarkPrice "The Plether oracle pays the Pyth fee from msg.value and refunds unused ETH to the caller or records a deferred claim if transfer fails" (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IOrderRouter.sol#L25-L27); IPerpsKeeper.executeLiquidation "The oracle refunds ETH above the Pyth fee to the caller or defers it as a caller-claimable balance" (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPerpsKeeper.sol#L71-L72); OrderRouter.executeLiquidation "The oracle handles Pyth fees and ETH refunds" (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L490).
Code: the router forwards exactly getUpdateFee() to the oracle and refunds the remainder itself (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L603-L608, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L621). A failed refund is credited to OrderRouterAdmin.claimableEth (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L274), never to PletherOracle.claimableEth.
ORACLE_SYNCHRONIZATION.md already describes both ledgers correctly (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L97-L98). A keeper contract written from the entrypoint NatSpec watches PletherOracle.claimableEth, sees 0, and does not know its refund is claimable from OrderRouterAdmin.claimBalance(). The PoC below shows it.
Suggested fix: update those NatSpec blocks to say the router refunds ETH above the Pyth fee and credits OrderRouterAdmin.claimableEth if the transfer fails.

3. The whitepaper says any complete pending deposit can be cancelled before maturity; Senior deposits cannot.

WHITEPAPER: "A complete pending deposit is cancellable before maturity" (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L698-L699).
Code: a pending Senior deposit cannot be cancelled unless one of the escape conditions holds (epoch rejection, settlement hold, projected terminal wipe, Senior impairment); otherwise cancelDepositRequest reverts with SeniorDepositCommitted (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L858).
The other docs agree with the code: WHITEPAPER L783-784, ACCOUNTING_SPEC L293-295 and SECURITY L878-880 call accepted Senior deposits firm commitments (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L783-L784, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L293-L295, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L878-L880).
Suggested fix: change WHITEPAPER L698 to "A complete pending Junior deposit is cancellable before maturity".

## Proof of Concept

## Proof of Concept


Items 1 and 3 are shown by the linked lines. Item 2, run against the unmodified repo (place the file in packages/perps/test/pocs_d4/):

FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/pocs_d4/D4_RefundLedger.t.sol -vv

Observed:

[PASS] test_D4_Control_EoaRefundedDirectly()
  EOA balance after refresh (wei) 10000000000000000
[PASS] test_D4_RefundDeferredToRouterAdminNotOracle()
  PletherOracle.claimableEth(keeper)     0
  OrderRouterAdmin.claimableEth(keeper)  10000000000000000
  ETH stranded in oracle (wei)           0
2 tests passed, 0 failed
solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// D4 (doc-vs-code, NatSpec): IOrderRouter.updateMarkPrice / OrderRouter.updateMarkPrice say the Plether ORACLE pays
// the Pyth fee from msg.value and refunds unused ETH to the caller "or records a deferred claim if transfer fails".
// Code (router/OrderLiquidationBatchLogic.sol:604-621) forwards exactly getUpdateFee() to the oracle and the ROUTER
// refunds the excess itself; a failed refund is credited to OrderRouterAdmin.claimableEth, never to
// PletherOracle.claimableEth. executeLiquidation / executeLiquidationBatch / settleLpEpoch share the same _sendEth.
// Value is not lost: RouterAdmin.claimBalance() pays it out. Informational NatSpec drift (ORACLE_SYNCHRONIZATION.md
// already tells consumers to expose both claim ledgers).

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

interface IRouterRefresh {

    function updateMarkPrice(
        bytes[] calldata pythUpdateData
    ) external payable;

}

interface IRouterAdminClaim {

    function claimBalance() external;

}

/// @dev Third-party keeper contract whose receive() rejects ETH until `accept` is flipped.
contract RejectingKeeper {

    bool public accept;

    function setAccept(
        bool a
    ) external {
        accept = a;
    }

    function refresh(
        address router,
        bytes[] calldata data
    ) external payable {
        IRouterRefresh(router).updateMarkPrice{value: msg.value}(data);
    }

    function claimFromRouterAdmin(
        address admin
    ) external {
        IRouterAdminClaim(admin).claimBalance();
    }

    receive() external payable {
        require(accept, "reject");
    }

}

contract D4_RefundLedger is BasePerpTest {

    uint256 internal constant PYTH_FEE = 1 gwei;
    uint256 internal constant EXCESS = 0.01 ether;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        vm.warp(SETUP_TIMESTAMP + 1 days);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setFee(PYTH_FEE);
    }

    function _ticks(
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory t = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            t[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(1e8), 0, -8, publishTime, publishTime - 1));
        }
        data = new bytes[](1);
        data[0] = abi.encode(t);
    }

    /// A: contract keeper rejecting ETH. Deferred refund lands in RouterAdmin, not in the oracle ledger.
    function test_D4_RefundDeferredToRouterAdminNotOracle() public {
        RejectingKeeper keeper = new RejectingKeeper();
        vm.deal(address(keeper), 0);
        bytes[] memory data = _ticks(vm.getBlockTimestamp());
        uint256 pythBefore = address(baseMockPyth).balance;

        keeper.refresh{value: PYTH_FEE + EXCESS}(address(router), data);

        uint256 oracleClaim = pletherOracle.claimableEth(address(keeper));
        uint256 adminClaim = routerAdmin.claimableEth(address(keeper));
        console2.log("pyth fee paid (wei)                  ", address(baseMockPyth).balance - pythBefore);
        console2.log("PletherOracle.claimableEth(keeper)    ", oracleClaim);
        console2.log("OrderRouterAdmin.claimableEth(keeper) ", adminClaim);
        console2.log("ETH stranded in oracle (wei)          ", address(pletherOracle).balance);

        assertEq(address(baseMockPyth).balance - pythBefore, PYTH_FEE, "exact Pyth fee paid");
        assertEq(oracleClaim, 0, "NatSpec says the oracle records the deferred claim; it records nothing");
        assertEq(adminClaim, EXCESS, "deferred refund is in the RouterAdmin ledger instead");
        assertEq(address(pletherOracle).balance, 0, "oracle never held the excess");

        // No value lost: the RouterAdmin ledger pays out.
        keeper.setAccept(true);
        keeper.claimFromRouterAdmin(address(routerAdmin));
        assertEq(address(keeper).balance, EXCESS, "excess recovered via RouterAdmin.claimBalance");
        assertEq(routerAdmin.claimableEth(address(keeper)), 0);
    }

    /// B (control): EOA keeper accepting ETH receives the excess directly; no ledger entry anywhere.
    function test_D4_Control_EoaRefundedDirectly() public {
        address eoa = address(0xE0A);
        vm.deal(eoa, PYTH_FEE + EXCESS);
        bytes[] memory data = _ticks(vm.getBlockTimestamp());
        vm.prank(eoa);
        router.updateMarkPrice{value: PYTH_FEE + EXCESS}(data);
        console2.log("EOA balance after refresh (wei)", eoa.balance);
        assertEq(eoa.balance, EXCESS, "excess refunded directly");
        assertEq(pletherOracle.claimableEth(eoa), 0);
        assertEq(routerAdmin.claimableEth(eoa), 0);
    }

}
