# \#98131 [SC-Low] Incomplete LOW-59 fix leaves CfdEngine ownership renounceable and can permanently freeze LP settlement

**Submitted on Nov 5th 2026 at 09:49:16 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98131
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

Brief/Intro

At the frozen commit, `CfdEngine` still exposes inherited `renounceOwnership()` even though the LOW-59 remediation removed that operation from three other recovery-critical perps contracts. Once the Engine owner is zero, a later ordinary degraded-mode latch cannot be cleared because the only recovery functions are `onlyOwner`; `HousePool` consequently rejects every LP epoch settlement, permanently preventing unfunded LP redemptions and pending deposit activations while LP assets remain in the pool.

## Vulnerability Details

### Affected version and incomplete LOW-59 fix

The affected production source is the competition pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The primary asset is `packages/perps/src/CfdEngine.sol`, included by the scope target **“All other contracts, libraries, router contracts and interfaces in packages/perps/src, except the six view-only contracts listed as out of scope.”** `HousePool.sol` and both `TrancheVault.sol` instances are affected downstream because LP settlement and redemption funding stop.

Cecuro LOW-59 identified that ownership renunciation could permanently remove emergency recovery authority. Its remediation required the project to “override `renounceOwnership()` in every governance-bearing perps contract.” Fix commit [`1151d01fd30d5fe6f6d80d604c5ddd4b90845dc6`](https://github.com/Plether-Fi/plether-core/commit/1151d01fd30d5fe6f6d80d604c5ddd4b90845dc6), dated 2026-10-20, instead added the override only to:

- `HousePool`
- `OrderRouterAdmin`
- `EmergencyPauseCoordinator`

The associated [PR #121](https://github.com/Plether-Fi/plether-core/pull/121), merged on 2026-10-21 as `e711621f8795909faa63f1abe4845550b88c4f1f`, describes the same three-contract resolution. `CfdEngine` was omitted even though its owner exclusively controls degraded-mode recovery and recapitalization. The competition rules expressly permit a same-root report on a fixed or acknowledged finding when the frozen fix is incomplete.

### Root cause in the frozen source

The following fragments are copied from the in-scope source and its pinned dependency at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

[`CfdTreasury`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L4-L10) inherits `Ownable2Step`, and [`CfdEngine`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L40-L46) inherits `CfdTreasury`:

```solidity
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step {
```

```solidity
contract CfdEngine is ICfdEngineTypes, IWithdrawGuard, ICfdEngineAdminHost, CfdTreasury, ReentrancyGuardTransient {
```

Neither contract overrides `renounceOwnership()`. The inherited OpenZeppelin implementation permanently transfers ownership to zero:

```solidity
function renounceOwnership() public virtual onlyOwner {
    _transferOwnership(address(0));
}
```

By contrast, the frozen [`HousePool` fix](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L260-L267) preserves recovery authority by reverting:

```solidity
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

The omitted Engine has the same recovery requirement. [`recapitalizePool()` and `clearDegradedMode()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L628-L667) are both `onlyOwner`, and the latter contains the only production write that clears the latch:

```solidity
function recapitalizePool(
    uint256 amountAssets
) external onlyOwner nonReentrant {
    // transfers owner funding into the configured pool
}

function clearDegradedMode() external onlyOwner {
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

Normal settlement enters degraded mode when available assets fall below the larger directional liability. [`CfdEngine._enterDegradedModeIfInsolvent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1368) performs that transition:

```solidity
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

Once the latch is set, [`HousePool._settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L840) checks it before reconciliation, redemption funding, deposit activation, or refund processing:

```solidity
function _settleLpEpoch(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot,
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot,
    uint256 cutoffEpoch
) internal returns (IHousePool.LpEpochSettlementResult memory result) {
    if (seniorVault == address(0) || juniorVault == address(0)) {
        revert HousePool__ZeroAddress();
    }
    _requireWithdrawalsLive(statusSnapshot);
    _requireFreshMark(accountingSnapshot, statusSnapshot);

    _reconcile(accountingSnapshot, statusSnapshot);
```

The guard is unconditional while the Engine snapshot reports degraded mode:

```solidity
function _requireWithdrawalsLive(
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
) internal pure {
    if (statusSnapshot.degradedMode) {
        revert HousePool__DegradedMode();
    }
}
```

The resulting failure is not tied to continuing insolvency. Even after permissionless position closes restore available assets above maximum liability, the boolean remains set until the missing Engine owner calls `clearDegradedMode()`.

### Preconditions

- The deployed `CfdEngine` uses the frozen source and is wired to `HousePool` and the two tranche vaults.
- The current Engine owner invokes the inherited `renounceOwnership()` operation that LOW-59 required every governance-bearing perps contract to disable.
- LP assets and at least one unfunded LP request or pending deposit activation exist when a later ordinary trader settlement enters degraded mode.
- Subsequent public trading or keeper activity can restore raw solvency, but cannot recreate Engine ownership or invoke an `onlyOwner` recovery function.

### Attack path

1. The `CfdEngine` owner calls the inherited `renounceOwnership()`. `owner()` becomes `address(0)`, and no pending owner exists.
2. LPs continue to have principal in the shared pool, and an LP redemption request reaches a settleable epoch.
3. A trader submits a normal close through `OrderRouter`; a permissionless keeper supplies the required Pyth update and executes it. The settlement leaves effective pool assets below maximum directional liability, so `_enterDegradedModeIfInsolvent()` sets `degradedMode = true`.
4. Further permissionless closes remove the remaining position liability and restore the Engine's solvency condition. This does not clear the boolean latch.
5. `clearDegradedMode()` and `recapitalizePool()` are now unreachable because both require the missing Engine owner. `transferOwnership()` is also unreachable, and `acceptOwnership()` has no pending owner to accept from.
6. Any caller that attempts LP epoch settlement reaches `_requireWithdrawalsLive()` and reverts with `HousePool__DegradedMode` before the pool can reconcile or fund matured redemption requests.
7. The same guard blocks both Senior and Junior epoch processing. Unfunded redemptions remain pending, pending deposit entries cannot activate, and the LP assets remain held by the existing pool.

### Why the freeze is permanent for the affected deployment

The scope states that the perps contracts are non-upgradeable. In addition, [`HousePool.ENGINE`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L129-L135) is immutable, and [`CfdEngine.setPool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L381-L400) is a one-time binding. The HousePool owner cannot clear Engine state or use the Engine-only payout path. Vault-authorized reconciliation also does not bypass `_requireWithdrawalsLive()`.

There is therefore no callable path in the affected deployment that changes `degradedMode` back to false, reassigns Engine ownership, or completes the blocked LP epoch after ownership has been renounced.

### Remediation

Override `renounceOwnership()` in `CfdEngine` and make it revert, matching the LOW-59 remediation already applied to `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`. Review every other governance-bearing perps contract against the same invariant rather than maintaining a contract-name allowlist.

Because the perps stack is non-upgradeable, deploy the corrected Engine and compatible pool/vault wiring before accepting user funds. Add regression coverage for the sequence `ownership renunciation -> degraded-mode entry -> solvency restored`, asserting that governance authority cannot be erased and that both LP tranches can resume epoch settlement.

## Impact Details

Selected in-scope impact: **Critical — Permanent freezing of funds**.

The affected security property is the ability of Senior and Junior LPs to convert matured withdrawal requests into funded claims through LP epoch settlement. After the Engine owner is zero and degraded mode is entered, every settlement attempt reverts before either tranche can be processed. Restoring economic solvency does not restore withdrawals because the recovery latch is controlled by an owner that can no longer exist.

The frozen value is LP principal and associated LP entitlement held by `HousePool`, not merely an unavailable interface or view. Unfunded redemption requests remain pending and pending LP deposits cannot activate. Already funded claims and trader close/liquidation paths are separate and do not repair the blocked LP settlement state.


## Proof of Concept

## Proof of Concept

PoC Explanation

The attachment is the single Solidity file `CfdEngineRenunciationPermanentFreezePoC.sol`. In a clean `immunefi-team/audit-comp-plether` checkout at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules initialized, place it at:

```text
packages/perps/test/perps/spec/admin/CfdEngineRenunciationPermanentFreezePoC.t.sol
```

From the repository root, run this command:

```bash
forge test --root packages/perps --match-path test/perps/spec/admin/CfdEngineRenunciationPermanentFreezePoC.t.sol -vv
```

The test file deploys the production contract topology without a protocol adapter. It uses the production `OrderRouter` V3 commit/execute lifecycle and release risk configuration. Only Pyth price delivery and the USDG settlement token are replaced with the repository's `MockPyth` and `MockSettlementToken`, as permitted by the competition PoC rules. No in-scope production contract is modified.

The fixture uses one Pyth feed to drive the Engine state transition. The release deployment uses six feeds, but the ownership transition, degraded-mode latch, solvency recovery, and LP settlement guard do not branch on feed count. All amounts, prices, percentages, addresses, gas measurements, and time advances below are controlled local test inputs rather than production observations.

The fixture deposits 3,000,000 six-decimal USDG into the Junior tranche and queues a 1,000,000 USDG-equivalent Junior redemption. It constructs a normal directional book through account-authored Router orders, then executes a full Short close after a local 10% mark change. The close latches degraded mode at effective assets `2,809,645.985931` USDG versus maximum liability `2,900,000` USDG. A later permissionless close removes the remaining position and restores raw solvency to effective assets `2,809,666.383515` USDG versus zero maximum liability.

The four tests separate the required causes:

1. `test_Control_OwnerPresent_RecoversAndFunds` keeps the Engine owner. After solvency is restored, the owner clears degraded mode and the matured redemption becomes funded with `935,910.153589` fixture USDG claimable.
2. `test_Control_RenouncedNoLatch_StillSettles` renounces Engine ownership without entering degraded mode. LP settlement still succeeds and produces `1,000,166.496698` fixture USDG claimable, showing that renunciation alone is not being mislabeled as the freeze.
3. `test_PoC_RenouncedLatchPermanentlyFreezesLpSettlement` renounces ownership before the ordinary close latches degraded mode. After raw solvency is restored, every caller still fails `clearDegradedMode()` and `recapitalizePool()`, every LP settlement reverts with `HousePool__DegradedMode`, the matured request remains pending, and its claimable amount remains zero after a 365-day local time advance. `HousePool` still contains `2,809,666.383515` fixture USDG.
4. `test_Trigger_OnlyOwnerCanRenounce` verifies that an outsider cannot invoke the trigger and that the current Engine owner can; the measured local gas cost is 31,558.

Expected terminal summary:

```text
Ran 4 tests for test/perps/spec/admin/CfdEngineRenunciationPermanentFreezePoC.t.sol:CfdEngineRenunciationPermanentFreezePoC
[PASS] test_Control_OwnerPresent_RecoversAndFunds()
[PASS] test_Control_RenouncedNoLatch_StillSettles()
[PASS] test_PoC_RenouncedLatchPermanentlyFreezesLpSettlement()
[PASS] test_Trigger_OnlyOwnerCanRenounce()
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

The file also contains an opt-in negative control. Running the probe with the false assumption that the ownerless latch eventually funds the redemption must return a nonzero exit code:

```bash
POC_NEGATIVE_CONTROL=true forge test --root packages/perps --match-path test/perps/spec/admin/CfdEngineRenunciationPermanentFreezePoC.t.sol --match-test test_PoC_RenouncedLatchPermanentlyFreezesLpSettlement -vv
```

Expected failing assertion:

```text
[FAIL: negative control: ownerless latch recovered unexpectedly: 0 <= 0]
Suite result: FAILED. 0 passed; 1 failed; 0 skipped
```

```
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {MockSettlementToken} from "../../../utils/MockSettlementToken.sol";

import {Test} from "forge-std/Test.sol";

/// @notice Reproduction of the incomplete LOW-59 fix in CfdEngine.
///
/// The fixture deploys the production topology only (no test adapter), uses the
/// DEPLOYMENT.md release risk parameters, a plain OrderRouter V3 commit/execute
/// lifecycle, and mocks only the out-of-scope Pyth feed and USDG token. The LP
/// is an unrelated third party and the timeline moves strictly forward.
///
/// Controls:
///  1. owner present: latch is recovered through the documented owner path;
///  2. renounced but no latch: settlement still funds the redemption, separating
///     renunciation from the degraded-mode latch;
///  3. renounced + latch: the latch is permanent, settlement reverts for a year,
///     and no pool/vault/owner path releases the redemption.
///
/// Set POC_NEGATIVE_CONTROL=true to enable an intentionally false recovery
/// assertion in the probe. Forge must then return a nonzero exit code.
contract CfdEngineRenunciationPermanentFreezePoC is Test {
    uint256 internal constant START_TIME = 1_709_532_000; // Monday 2024-03-04 10:00 UTC (outside FAD)
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 50; // DEPLOYMENT.md frozen-close spread
    uint256 internal constant JUNIOR_DEPOSIT = 3_000_000e6;
    uint256 internal constant LONG_SIZE = 1_000_000e18;
    uint256 internal constant SHORT_SIZE = 2_100_000e18;
    uint256 internal constant LONG_ADD = 1_900_000e18;
    uint256 internal constant TRADER_FUNDING = 250_000e6;
    uint256 internal constant REDEEM_ASSETS = 1_000_000e6;
    uint256 internal constant VOLATILITY_MARK = 110_000_000; // +10% basket move

    bytes32 internal constant FEED = bytes32(uint256(1));

    address internal constant SHORT_TRADER = address(0xA11CE);
    address internal constant LONG_TRADER = address(0xB0B);
    address internal constant KEEPER = address(0xCAFE);
    address internal constant OUTSIDER = address(0xBAD);
    address internal constant LP = address(0x1F1F);

    MockSettlementToken internal settlementToken;
    MockPyth internal pyth;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    OrderRouter internal router;

    uint256 internal clientNonce;

    /// @dev DEPLOYMENT.md release parameters at the frozen commit.
    function _releaseRiskParams() internal pure returns (CfdTypes.RiskParams memory) {
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

    function setUp() public {
        vm.warp(START_TIME);

        settlementToken = new MockSettlementToken();
        clearinghouse = new MarginClearinghouse(address(settlementToken));

        engine = new CfdEngine(
            address(settlementToken), address(clearinghouse), CAP_PRICE, _releaseRiskParams(), FROZEN_CLOSE_SPREAD_BPS
        );
        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));
        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(CAP_PRICE))));
        engine.setProtocolTreasury(address(0xFEE5));

        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
        seniorVault =
            new TrancheVault(settlementToken, address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0));
        juniorVault =
            new TrancheVault(settlementToken, address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0));
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        pyth = new MockPyth();
        bytes32[] memory feeds = new bytes32[](1);
        feeds[0] = FEED;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 1e18;
        uint256[] memory bases = new uint256[](1);
        bases[0] = PRICE;
        PletherOracle oracle =
            new PletherOracle(address(engine), address(pool), address(pyth), feeds, weights, bases, new bool[](1));

        CfdEngineLens engineLens = new CfdEngineLens(address(engine));
        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar executionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, address(engine), address(clearinghouse), address(pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        router = new OrderRouter(
            address(engine),
            address(engineLens),
            address(pool),
            address(oracle),
            address(keeperSidecar),
            address(evaluator),
            address(executionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(router), predictedRouter, "router address prediction");
        engine.setOrderRouter(address(router));
        clearinghouse.setEngine(address(engine));

        // Activate the DEPLOYMENT.md release pool configuration (40M USDG / 80% Senior cap)
        // through the normal 72h owner timelock. Time only moves forward afterwards.
        IHousePool.PoolConfig memory config = IHousePool.PoolConfig({
            seniorRateBps: pool.seniorRateBps(),
            markStalenessLimit: pool.markStalenessLimit(),
            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
            maxSeniorExposureAssets: 40_000_000e6,
            maxSeniorShareBps: 8_000
        });
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();

        // Bootstrap both tranche seeds and activate trading (documented owner actions).
        settlementToken.mint(address(this), 2_000e6);
        settlementToken.approve(address(pool), 2_000e6);
        pool.initializeSeedPosition(false, 1_000e6, address(this));
        pool.initializeSeedPosition(true, 1_000e6, address(this));
        pool.activateTrading();

        // Third-party Junior LP supplies the whole pool depth.
        settlementToken.mint(LP, JUNIOR_DEPOSIT);
        vm.startPrank(LP);
        settlementToken.approve(address(juniorVault), JUNIOR_DEPOSIT);
        uint256 depositId = juniorVault.requestDeposit(JUNIOR_DEPOSIT, LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(depositId));
        pool.settleLpEpoch(0, 0);
        vm.startPrank(LP);
        uint256 claimableDeposit = juniorVault.claimableDepositRequest(depositId, LP);
        juniorVault.claimDeposit(depositId, claimableDeposit, LP, LP);
        vm.stopPrank();

        _fundTrader(SHORT_TRADER);
        _fundTrader(LONG_TRADER);
    }

    // ------------------------------------------------------------------
    // Control 1: owner present, documented recovery funds the redemption
    // ------------------------------------------------------------------

    function test_Control_OwnerPresent_RecoversAndFunds() public {
        uint256 redeemId = _runScenarioToLatch(false);

        assertEq(engine.owner(), address(this), "owner present");
        assertTrue(engine.degradedMode(), "latch set");

        _keeperCloseLongAtEntry();
        (uint256 effectiveAssets, uint256 maxLiability) = _rawSolvency();
        emit log_named_uint("control_effective_after_recovery", effectiveAssets);
        emit log_named_uint("control_liability_after_recovery", maxLiability);
        assertGe(effectiveAssets, maxLiability, "solvency restored");
        assertTrue(engine.degradedMode(), "latch needs the owner");

        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner clears");
        assertTrue(pool.isWithdrawalLive(), "withdrawals live");

        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableRedeemAssets(redeemId, LP);
        emit log_named_uint("control_claimable", claimable);
        assertEq(juniorVault.pendingRedeemRequest(redeemId, LP), 0, "redemption funded");
        assertGe(claimable, REDEEM_ASSETS * 90 / 100, "material redemption at post-loss NAV");
    }

    // ------------------------------------------------------------------
    // Control 2: renounced but no latch -> settlement still works
    // ------------------------------------------------------------------

    function test_Control_RenouncedNoLatch_StillSettles() public {
        uint256 redeemId = _queueMaturedJuniorRedeem();
        _markRefresh(PRICE);

        // Small, fully collateralised book; both sides closed flat at the entry mark.
        _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, 500_000e18, 10_000e6, PRICE, false, PRICE);
        _commitAndExecute(SHORT_TRADER, CfdTypes.Side.SHORT, 300_000e18, 10_000e6, PRICE, false, PRICE);
        _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, 500_000e18, 0, type(uint256).max, true, PRICE);
        _commitAndExecute(SHORT_TRADER, CfdTypes.Side.SHORT, 300_000e18, 0, 1, true, PRICE);

        engine.renounceOwnership();
        assertEq(engine.owner(), address(0), "engine owner renounced");
        assertFalse(engine.degradedMode(), "ordinary small book does not latch");

        _warpTo(pool.lpEpochStart(redeemId));
        _markRefresh(PRICE);
        pool.settleLpEpoch(0, 0);

        uint256 claimable = juniorVault.claimableRedeemAssets(redeemId, LP);
        emit log_named_uint("control_no_latch_claimable", claimable);
        assertEq(juniorVault.pendingRedeemRequest(redeemId, LP), 0, "redemption funded");
        assertGe(claimable, REDEEM_ASSETS * 95 / 100, "material redemption funded without an engine owner");
    }

    // ------------------------------------------------------------------
    // Probe: renounced + ordinary latch -> permanent LP freeze
    // ------------------------------------------------------------------

    function test_PoC_RenouncedLatchPermanentlyFreezesLpSettlement() public {
        uint256 redeemId = _runScenarioToLatch(true);

        // Renunciation succeeded and is one-way.
        assertEq(engine.owner(), address(0), "engine owner is zero");
        assertEq(engine.pendingOwner(), address(0), "no pending owner");
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, OUTSIDER));
        vm.prank(OUTSIDER);
        engine.transferOwnership(OUTSIDER);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, OUTSIDER));
        vm.prank(OUTSIDER);
        engine.acceptOwnership();

        // The latch arose from an ordinary permissionless close and is already set.
        assertTrue(engine.degradedMode(), "ordinary close latched degraded mode");
        assertGt(juniorVault.pendingRedeemRequest(redeemId, LP), 0, "third-party redemption queued");

        // Permissionless keepers restore raw solvency but cannot clear the latch.
        _keeperCloseLongAtEntry();
        (uint256 effectiveAssets, uint256 maxLiability) = _rawSolvency();
        emit log_named_uint("poc_effective_after_recovery", effectiveAssets);
        emit log_named_uint("poc_liability_after_recovery", maxLiability);
        assertGe(effectiveAssets, maxLiability, "raw solvency genuinely restored");
        assertTrue(engine.degradedMode(), "latch stays set without an owner");

        // No caller can clear the latch or recapitalize.
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, OUTSIDER));
        vm.prank(OUTSIDER);
        engine.clearDegradedMode();
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.recapitalizePool(1_000_000e6);

        // Every LP settlement reverts before reconcile/redemption funding.
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        pool.settleLpEpoch(0, 0);
        vm.warp(block.timestamp + 365 days);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        pool.settleLpEpoch(0, 0);

        // The redemption stays unfunded and all LP gates report closed.
        assertGt(juniorVault.pendingRedeemRequest(redeemId, LP), 0, "request still pending after a year");
        assertEq(juniorVault.claimableRedeemAssets(redeemId, LP), 0, "nothing claimable");
        assertFalse(pool.isWithdrawalLive(), "withdrawal gate closed");
        assertFalse(pool.canAcceptTrancheDeposits(false), "junior entry gate closed");
        assertFalse(pool.canSettleDepositEntries(), "deposit activation gate closed");

        // The HousePool owner still exists but cannot cure the engine latch, pay out
        // accounted cash, or renounce its own (patched) ownership.
        assertEq(pool.owner(), address(this), "pool owner present");
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        pool.renounceOwnership();
        vm.expectRevert(IHousePool.HousePool__Unauthorized.selector);
        pool.payOut(LP, 1);

        // Vault-authorised reconcile is not a release path.
        vm.prank(address(juniorVault));
        pool.reconcile();
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        pool.settleLpEpoch(0, 0);

        uint256 trapped = pool.totalAssets();
        uint256 claimableAfterOneYear = juniorVault.claimableRedeemAssets(redeemId, LP);
        emit log_named_uint("poc_trapped_pool_assets", trapped);
        emit log_named_uint("poc_lp_claimable_after_one_year", claimableAfterOneYear);
        assertGt(trapped, REDEEM_ASSETS, "LP principal remains trapped in the pool");

        if (vm.envOr("POC_NEGATIVE_CONTROL", false)) {
            assertGt(claimableAfterOneYear, 0, "negative control: ownerless latch recovered unexpectedly");
        }
    }

    // ------------------------------------------------------------------
    // Trigger measurement: only the owner can renounce, and its cost
    // ------------------------------------------------------------------

    function test_Trigger_OnlyOwnerCanRenounce() public {
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, OUTSIDER));
        vm.prank(OUTSIDER);
        engine.renounceOwnership();

        uint256 gasBefore = gasleft();
        engine.renounceOwnership();
        uint256 gasUsed = gasBefore - gasleft();
        emit log_named_uint("renounce_gas", gasUsed);
        assertEq(engine.owner(), address(0), "renunciation succeeded");
        assertLt(gasUsed, 100_000, "renunciation is a single cheap call");
    }

    // ------------------------------------------------------------------
    // Scenario
    // ------------------------------------------------------------------

    function _runScenarioToLatch(bool renounceFirst) internal returns (uint256 redeemId) {
        redeemId = _queueMaturedJuniorRedeem();
        _buildDirectionalBook();
        assertFalse(engine.degradedMode(), "solvent before the volatility event");

        if (renounceFirst) {
            engine.renounceOwnership();
            assertEq(engine.owner(), address(0), "engine ownerless before the market event");
        }

        // Ordinary market move: +10% basket over 15 minutes. A permissionless keeper
        // publishes the new mark and executes the account-authored close at that mark.
        vm.warp(block.timestamp + 15 minutes);
        _markRefresh(VOLATILITY_MARK);
        (, OrderV3Types.ExecutionResult memory closeResult) =
            _commitAndExecute(SHORT_TRADER, CfdTypes.Side.SHORT, SHORT_SIZE, 0, 1, true, VOLATILITY_MARK);
        assertEq(uint8(closeResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "short close executed");
        assertTrue(engine.degradedMode(), "ordinary close latches degraded mode");

        (uint256 effectiveAtLatch, uint256 liabilityAtLatch) = _rawSolvency();
        emit log_named_uint("effective_assets_at_latch", effectiveAtLatch);
        emit log_named_uint("max_liability_at_latch", liabilityAtLatch);
        assertLt(effectiveAtLatch, liabilityAtLatch, "latch requires E < L");

        _warpTo(pool.lpEpochStart(redeemId));
    }

    function _buildDirectionalBook() internal {
        _markRefresh(PRICE);

        (, OrderV3Types.ExecutionResult memory openLong) =
            _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, LONG_SIZE, 10_000e6, PRICE, false, PRICE);
        assertEq(uint8(openLong.status), uint8(OrderV3Types.LifecycleStatus.Executed), "LONG open executed");

        (, OrderV3Types.ExecutionResult memory openShort) =
            _commitAndExecute(SHORT_TRADER, CfdTypes.Side.SHORT, SHORT_SIZE, 10_000e6, PRICE, false, PRICE);
        assertEq(uint8(openShort.status), uint8(OrderV3Types.LifecycleStatus.Executed), "SHORT open executed");

        (, OrderV3Types.ExecutionResult memory addLong) =
            _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, LONG_ADD, 10_000e6, PRICE, false, PRICE);
        assertEq(uint8(addLong.status), uint8(OrderV3Types.LifecycleStatus.Executed), "LONG increase executed");

        (uint256 longSize,,,,,,) = engine.positions(LONG_TRADER);
        (uint256 shortSize,,,,,,) = engine.positions(SHORT_TRADER);
        assertEq(longSize, LONG_SIZE + LONG_ADD, "long book");
        assertEq(shortSize, SHORT_SIZE, "short book");
    }

    function _queueMaturedJuniorRedeem() internal returns (uint256 requestId) {
        vm.warp(block.timestamp + 1 hours + 1);
        uint256 shares = juniorVault.estimateWithdrawShares(REDEEM_ASSETS);
        vm.prank(LP);
        requestId = juniorVault.requestRedeem(shares, LP, LP);
    }

    function _keeperCloseLongAtEntry() internal {
        // The basket returns to the entry price; a permissionless keeper closes flat.
        vm.warp(block.timestamp + 15 minutes);
        _markRefresh(PRICE);
        (, OrderV3Types.ExecutionResult memory closeResult) =
            _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, LONG_SIZE + LONG_ADD, 0, type(uint256).max, true, PRICE);
        assertEq(uint8(closeResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "long close executed");
        (uint256 size,,,,,,) = engine.positions(LONG_TRADER);
        assertEq(size, 0, "long position flat");
    }

    // ------------------------------------------------------------------
    // Order lifecycle helpers (production V3 path)
    // ------------------------------------------------------------------

    function _commitAndExecute(
        address account,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose,
        uint256 executionPrice
    ) internal returns (uint64 orderId, OrderV3Types.ExecutionResult memory result) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("cfd-engine-renunciation-poc", clientNonce++));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = targetPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });

        vm.prank(account);
        orderId = router.commitOrder(request);

        uint64 commitTime = uint64(block.timestamp);
        bytes[] memory update = _pythUpdate(executionPrice, commitTime);

        vm.prank(KEEPER);
        result = router.executeOrder(orderId, update);
    }

    function _markRefresh(uint256 price) internal {
        bytes[] memory update = _pythUpdate(price, block.timestamp);
        router.updateMarkPrice(update);
    }

    /// @dev Explicit MockPyth FeedUpdate payload modelling a signed Pyth update
    /// published after the commit. The mock replaces only the out-of-scope Pyth feed.
    function _pythUpdate(uint256 price, uint256 previousPublishTime) internal returns (bytes[] memory update) {
        uint256 publishTime = block.timestamp + 2;
        vm.warp(publishTime);
        vm.roll(block.number + 1);

        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](1);
        updates[0] = MockPyth.FeedUpdate({
            id: FEED,
            price: MockPyth.MockPrice({
                price: int64(uint64(price)),
                conf: 0,
                expo: -8,
                publishTime: publishTime,
                prevPublishTime: previousPublishTime
            })
        });
        update = new bytes[](1);
        update[0] = abi.encode(updates);
    }

    // ------------------------------------------------------------------
    // Funding and views
    // ------------------------------------------------------------------

    function _fundTrader(address trader) internal {
        settlementToken.mint(trader, TRADER_FUNDING);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), TRADER_FUNDING);
        clearinghouse.deposit(trader, TRADER_FUNDING);
        vm.stopPrank();
    }

    function _warpTo(uint256 timestamp) internal {
        if (block.timestamp < timestamp) {
            vm.warp(timestamp);
        }
    }

    /// @dev Mirrors CfdEngine._availableCashForFreshPoolPayouts() and _maxLiability() on public state.
    function _rawSolvency() internal view returns (uint256 effectiveAssets, uint256 maxLiability) {
        (uint256 longMaxProfit,,,) = engine.sides(0);
        (uint256 shortMaxProfit,,,) = engine.sides(1);
        maxLiability = longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;

        uint256 poolAssets = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        effectiveAssets = poolAssets > claims ? poolAssets - claims : 0;
    }
}
```
