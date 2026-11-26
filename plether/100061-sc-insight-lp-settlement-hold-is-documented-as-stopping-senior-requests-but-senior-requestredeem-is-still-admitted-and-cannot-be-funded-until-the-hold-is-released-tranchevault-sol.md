# \#100061 [SC-Insight] LP settlement hold is documented as stopping Senior requests, but Senior `requestRedeem` is still admitted and cannot be funded until the hold is released (`TrancheVault.sol`)

**Submitted on Nov 16th 2026 at 07:04:19 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100061
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

## Brief/Intro

The LP settlement hold (`HousePool.pauseLpEpochSettlement`) is the guardian's emergency containment tool. The interface NatSpec, the HousePool NatSpec and SECURITY.md all describe it as stopping new Senior requests. In the code only Senior *deposit* requests and reservations check the hold. `TrancheVault.requestRedeem` has no hold check, so a Senior LP can still request a redemption during the hold. The shares move into a pending request that cannot be funded, because `settleLpEpoch` reverts until the owner releases the hold, and the hold has no automatic expiry.

## Vulnerability Details

The docs describe the hold as stopping Senior requests:

- `packages/perps/src/interfaces/IHousePool.sol:288`: "Senior requests stop and existing Senior commitments become cancellable while the hold is active."
- `packages/perps/src/HousePool.sol:398`: "Holds new LP epoch clearing and Senior request admission without pausing Junior requests or trading."
- `SECURITY.md:497-499`: "It stops new Senior requests and reservations while allowing existing Senior commitments to be refunded. Junior requests, redemptions, trading, recapitalization/reconciliation, and already-funded claims remain live."
- `SECURITY.md:1155-1157`: the hold "also blocks new Senior requests and reservations ... Junior deposit and redemption requests, reconciliation, and funded claims remain live." Listing Junior redemption requests as staying live implies that Senior redemption requests do not.

The only admission check is on the deposit side:

```solidity
// packages/perps/src/HousePool.sol#L483  (_canAcceptTrancheDeposits)
        if (isSenior && lpEpochSettlementPaused) {
```

`TrancheVault.requestRedeem` (`TrancheVault.sol:606-640`) and `requestRedeemFromClaimableDeposit` never read `lpEpochSettlementPaused`. The only hold check in `TrancheVault.sol` is on deposit capacity (`:853`).

The documents also disagree with each other: `EMERGENCY_RESPONSE_GUIDE.md:77` limits the stop to "Request a Senior deposit / reserve capacity", while SECURITY.md and the NatSpec say "Senior requests". A reader cannot tell from the docs whether Senior redemption requests are admitted.

## Impact Details

- Operators reading the NatSpec during an incident will assume that no new Senior queue entries can form while the hold is active. In practice Senior redemption demand keeps building and is all processed at the first epoch after release, which changes the redemption-priority picture governance sees while it decides how to recover.
- A Senior LP who requests during the hold can cancel only until the request's epoch starts (`TrancheVault.sol:894`, `currentLpEpoch() >= requestId`; epochs are 1 hour). After that the shares stay escrowed in a request that cannot be funded until the owner releases the hold.

No funds are lost; this is a documentation and specification mismatch, submitted as an Insight.

## Recommendation

Either gate Senior `requestRedeem` and `requestRedeemFromClaimableDeposit` on `lpEpochSettlementPaused`, or reword `IHousePool.lpEpochSettlementPaused`, `HousePool.pauseLpEpochSettlement` and SECURITY.md to say "Senior deposit requests and reservations", and state that Senior redeem requests are admitted but cannot be funded during the hold.

## References

- `packages/perps/src/TrancheVault.sol#L606-L640`, `packages/perps/src/HousePool.sol#L398`, `#L483`, `packages/perps/src/interfaces/IHousePool.sol#L288` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/SECURITY.md:497-499`, `:1155-1157`; `EMERGENCY_RESPONSE_GUIDE.md:77`

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the in-scope commit. No in-scope contract is modified; only Pyth (`MockPyth`) and USDG are mocked. `PocBase.sol` loads the DEPLOYMENT.md release parameters. The full source of every file needed is included below.

**Setup:**
```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the 2 files below into packages/perps/test/perps/audit/
forge test --root packages/perps --match-path "test/perps/audit/INS3_SeniorRedeemDuringSettlementHold.t.sol" -vv
```

**Output:**
```
[PASS] test_Control_SeniorDepositRequestBlockedDuringHold() (gas: 268862)
Logs:
  0x15fb1002
[PASS] test_SeniorRedeemRequestAdmittedDuringHold() (gas: 442191)
Logs:
  senior redeem request id during hold 474947
  shares escrowed 9993155259834
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

- `test_SeniorRedeemRequestAdmittedDuringHold`: with the hold active, a Senior redeem request is created and all of the LP's shares (9,993,155.26 shares) move into it; one day later `settleLpEpoch` reverts `HousePool__LpEpochSettlementPaused`.
- `test_Control_SeniorDepositRequestBlockedDuringHold`: `canAcceptTrancheDeposits(true) == false`, and a Senior `requestDeposit` reverts (`0x15fb1002`, `TrancheVault__DepositsUnavailable`).

### `packages/perps/test/perps/audit/INS3_SeniorRedeemDuringSettlementHold.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.sol";
import {console2} from "forge-std/console2.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice IHousePool NatSpec ("Senior requests stop ... while the hold is active") and HousePool.sol:398
///         ("Holds ... Senior request admission") say the LP settlement hold stops Senior requests. In code only the
///         Senior DEPOSIT request is gated (HousePool._canAcceptTrancheDeposits); TrancheVault.requestRedeem has no
///         hold check, so new Senior redeem requests are admitted (and escrow shares) while no epoch can settle.
contract INS3_SeniorRedeemDuringSettlementHoldTest is PocBase {

    uint256 internal constant AMOUNT = 10_000e6;

    function setUp() public override {
        super.setUp();
        _fundSenior(lpAlice, AMOUNT);
        vm.warp(block.timestamp + seniorVault.DEPOSIT_COOLDOWN() + 1);
        pool.pauseLpEpochSettlement(); // owner = this test
        assertTrue(pool.lpEpochSettlementPaused(), "hold active");
    }

    function test_SeniorRedeemRequestAdmittedDuringHold() public {
        uint256 shares = seniorVault.balanceOf(lpAlice);
        assertGt(shares, 0);
        vm.prank(lpAlice);
        uint256 requestId = seniorVault.requestRedeem(shares, lpAlice, lpAlice);
        console2.log("senior redeem request id during hold", requestId);
        console2.log("shares escrowed", shares - seniorVault.balanceOf(lpAlice));
        assertEq(seniorVault.balanceOf(lpAlice), 0, "shares moved into the request");
        assertGt(seniorVault.pendingRedeemRequest(requestId, lpAlice), 0, "request pending");
        // The request cannot be funded while the hold lasts: the canonical epoch transition reverts.
        vm.warp(block.timestamp + 1 days);
        vm.expectRevert(IHousePool.HousePool__LpEpochSettlementPaused.selector);
        pool.settleLpEpoch(0, 0);
    }

    function test_Control_SeniorDepositRequestBlockedDuringHold() public {
        assertFalse(pool.canAcceptTrancheDeposits(true), "senior entry gate closed");
        settlementToken.mint(lpAlice, AMOUNT);
        vm.startPrank(lpAlice);
        settlementToken.approve(address(seniorVault), AMOUNT);
        bytes4 sel;
        try seniorVault.requestDeposit(AMOUNT, lpAlice, lpAlice) {
            revert("senior deposit request admitted during hold");
        } catch (bytes memory e) {
            sel = bytes4(e);
        }
        vm.stopPrank();
        console2.logBytes4(sel);
        // The hold closes the Senior entry gate (HousePool.sol:483 via canAcceptTrancheDeposits).
        assertEq(sel, TrancheVault.TrancheVault__DepositsUnavailable.selector, "senior deposits unavailable during the hold");
    }

}
```

### `packages/perps/test/perps/audit/PocBase.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// =====================================================================================================================
// Immunefi Plether audit-comp PoC base (local only, not part of the upstream suite).
//
// Why this exists instead of using BasePerpTest directly:
//   * BasePerpTest's `_open/_close` helpers `vm.prank(address(router))` straight into the engine. That bypasses every
//     router/policy check, so a PoC built on them is not a valid user-reachable exploit. The helpers here go through
//     the production `OrderRouter.commitOrder(OrderV3Types.OrderRequest)` + `executeOrder` surface instead.
//   * BasePerpTest's risk params (1% MM / 1.5% IM, no VPI) differ from the Arbitrum deployment params Immunefi uses
//     for severity. This base installs the deployment values from script/DeployPerpsArbitrumSepolia.s.sol.
//
// Rules reminder (Immunefi scope page): do NOT modify in-scope contracts; mock only out-of-scope pieces (Pyth, USDG).
// Owner/Pauser/Guardian/treasury are trusted. Keepers are NOT trusted (any address may act as keeper).
//
// Run:  forge test --root packages/perps --match-path "test/perps/audit/*" -vvv
// =====================================================================================================================

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

abstract contract PocBase is BasePerpTest {

    // --- Deployment parameters (script/DeployPerpsArbitrumSepolia.s.sol, v2.0.0) ---
    uint256 internal constant DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant DEPLOY_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS = 2500;

    /// @dev Position sizes must be multiples of CfdTypes.SIZE_QUANTUM (1e20). Price is 8 decimals, USDG 6 decimals.
    uint256 internal constant ONE_DOLLAR = 1e8;

    address internal keeper = makeAddr("keeper");
    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");
    address internal lpAlice = makeAddr("lpAlice");

    // ---------------------------------------------------------------------------------------------------------------
    // Setup
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Deployment risk params (constructor-installed, so no timelock needed).
    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    /// @dev 1M junior + 1M senior LP capital on top of the 1k/1k seed positions.
    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 1_000_000e6;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 1_000_000e6;
    }

    function setUp() public virtual override {
        super.setUp();
        _applyDeploymentTimelockedConfig();
        vm.label(address(engine), "CfdEngine");
        vm.label(address(pool), "HousePool");
        vm.label(address(clearinghouse), "MarginClearinghouse");
        vm.label(address(router), "OrderRouter");
        vm.label(address(seniorVault), "SeniorVault");
        vm.label(address(juniorVault), "JuniorVault");
        vm.label(address(pletherOracle), "PletherOracle");
        vm.label(address(settlementToken), "USDG(mock)");
    }

    /// @dev Senior caps and the oracle confidence multiplier are timelocked (72h). Propose both, warp once, finalize.
    ///      Setup starts Monday 10:00 UTC, so +72h lands on Thursday (outside the FAD/weekend window).
    function _applyDeploymentTimelockedConfig() internal {
        IHousePool.PoolConfig memory poolCfg = _currentPoolConfig();
        poolCfg.maxSeniorExposureAssets = DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS;
        poolCfg.maxSeniorShareBps = DEPLOY_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(poolCfg);

        IOrderRouterAdminHost.RouterConfig memory routerCfg = _routerConfig();
        routerCfg.adverseConfidenceMultiplierBps = DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS;
        routerAdmin.proposeRouterConfig(routerCfg);

        uint256 activation = pool.poolConfigActivationTime();
        vm.warp(activation > block.timestamp + 72 hours + 1 ? activation : block.timestamp + 72 hours + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Order helpers: production V3 surface only
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Permissive bounds; tighten individual fields in a PoC when the bound itself is under test.
    function _permissiveBounds() internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        b.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        b.allowedExecutionModes = 7;
        b.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.minPostSettlementBalanceAssets = 0;
        b.minPostPositionEquityAssets = 0;
        b.maxPostLeverageBps = type(uint32).max;
    }

    /// @dev Unbounded slippage target, as used by the legacy harness: LONG profits when the basket falls.
    function _anyPriceTarget(
        CfdTypes.Side side,
        bool isClose
    ) internal pure returns (uint256) {
        return (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory req) {
        req.clientOrderId = keccak256(abi.encode("poc", router.nextCommitId(), side, size, margin, isClose));
        req.side = side;
        req.sizeDelta = size;
        req.marginDelta = isClose ? 0 : margin;
        req.targetPrice = _anyPriceTarget(side, isClose);
        req.isClose = isClose;
        req.bounds = _permissiveBounds();
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory req
    ) internal returns (uint64 id) {
        vm.prank(trader);
        id = router.commitOrder(req);
    }

    /// @dev Publishes a fresh mock Pyth price (advances block+1s) and executes `id` as `executor`.
    function _execute(
        uint64 id,
        uint256 price,
        address executor
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(executor);
        result = router.executeOrder(id, update);
    }

    function _openViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 id = _commit(trader, _request(side, size, margin, false));
        result = _execute(id, price, keeper);
    }

    function _closeViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 id = _commit(trader, _request(side, size, 0, true));
        result = _execute(id, price, keeper);
    }

    function _liquidateViaRouter(
        address account,
        uint256 price,
        address executor
    ) internal {
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(executor);
        router.executeLiquidation(account, update);
    }

    function _assertExecuted(
        OrderV3Types.ExecutionResult memory result,
        string memory label
    ) internal pure {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), label);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Actors & funding
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Mints mock USDG to `trader` and deposits it into the clearinghouse (free settlement, no position margin).
    function _fundTraderCash(
        address trader,
        uint256 amount
    ) internal {
        _fundTrader(trader, amount);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Accounting snapshots: use these to show "who lost what" in a PoC
    // ---------------------------------------------------------------------------------------------------------------

    struct Snap {
        uint256 clearinghouseUsdg;
        uint256 poolUsdg;
        uint256 poolTotalAssets;
        uint256 seniorTotalAssets;
        uint256 juniorTotalAssets;
        uint256 attackerCustody;
        uint256 attackerWallet;
        uint256 victimCustody;
        uint256 attackerClaim;
    }

    function _snap() internal view returns (Snap memory s) {
        s.clearinghouseUsdg = settlementToken.balanceOf(address(clearinghouse));
        s.poolUsdg = settlementToken.balanceOf(address(pool));
        s.poolTotalAssets = pool.totalAssets();
        s.seniorTotalAssets = seniorVault.totalAssets();
        s.juniorTotalAssets = juniorVault.totalAssets();
        s.attackerCustody = clearinghouse.balanceAssets(attacker);
        s.attackerWallet = settlementToken.balanceOf(attacker);
        s.victimCustody = clearinghouse.balanceAssets(victim);
        s.attackerClaim = engine.traderClaimBalanceAssets(attacker);
    }

    function _logSnap(
        string memory label,
        Snap memory s
    ) internal pure {
        console2.log("==", label);
        console2.log("  clearinghouse USDG ", s.clearinghouseUsdg);
        console2.log("  pool USDG          ", s.poolUsdg);
        console2.log("  pool totalAssets   ", s.poolTotalAssets);
        console2.log("  senior totalAssets ", s.seniorTotalAssets);
        console2.log("  junior totalAssets ", s.juniorTotalAssets);
        console2.log("  attacker custody   ", s.attackerCustody);


        console2.log("  junior totalAssets ", s.juniorTotalAssets);
        console2.log("  attacker custody   ", s.attackerCustody);
        console2.log("  attacker wallet    ", s.attackerWallet);
    }

    /// @dev Total USDG held by protocol contracts. Useful conservation check: value can move between buckets but an
    ///      exploit typically shows this falling while an attacker's wallet rises by more than they put in.
    function _protocolUsdg() internal view returns (uint256) {
        return settlementToken.balanceOf(address(clearinghouse)) + settlementToken.balanceOf(address(pool))
            + settlementToken.balanceOf(address(seniorVault)) + settlementToken.balanceOf(address(juniorVault))
            + settlementToken.balanceOf(address(engine)) + settlementToken.balanceOf(address(router));
    }

    // Fix for F05_OrderExpiryTest: executeExt is called but not defined
    // Adding stub implementation
    function executeExt(uint64 cid, uint256 px) external virtual {
    }

}
```

