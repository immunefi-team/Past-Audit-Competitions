# \#99905 [SC-Medium] A reversible terminal mark lets wiped Junior shares capture quarantined recovery assets

**Submitted on Nov 15th 2026 at 17:32:23 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99905
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Description

## Brief/Intro

`unassignedAssets` is the protocol's ownership quarantine for canonical HousePool USDG that does not currently have a safe tranche claimant. It is excluded from Senior/Junior ownership and ordinary withdrawals until governance explicitly assigns it and mints matching shares. Accordingly, this report does **not** characterize the affected amount as user principal or unclaimed yield; the demonstrated harm is unauthorized capture of an ownerless protocol recovery reserve.

Reconciliation nevertheless caps the **stored ownership ledger** by terminal distributable equity after unrealized open-position PnL:

```solidity
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

Applying a temporary cap for marked NAV and withdrawal safety can be reasonable. The issue is that the cap overwrites the **persistent source-ownership ledger** even though the mark causes no position close, trader claim, USDG payout or other realized consumption. The reduction has no restoration ledger. When the same positions later close at their entry price, the physical USDG is still present but the smaller stored reserve is treated as authoritative; the difference is routed through the ordinary waterfall and becomes redeemable by existing Junior shares.

The PoC first reaches a legitimate recovery state through public trading and the documented `recapitalizePool` action. Governance restores the exact Senior deficit plus a 40,000-USDG surplus. This uses the same surplus-quarantine path exercised by the repository's own `EngineRecapitalizationTest`; the larger amount ensures that the reclassified Junior value exceeds the release 80% Senior-share covenant and can be partially withdrawn.

An attacker then:

1. controls wiped Junior shares whose protocol NAV is zero (the PoC models acquisition from the wiped holder for 1 USDG);
2. opens a balanced LONG/SHORT book through public V3 orders;
3. persists one temporary terminal mark through permissionless LP settlement;
4. closes both positions at their original entry, realizing no price PnL; and
5. redeems the value reclassified out of `unassignedAssets`.

The attack removes **31,903.649847 USDG** from the persistent quarantine accounting. The release 80% Senior-share covenant retains most of the resulting Junior value as required backing, but still permits an actual **7,964.297887-USDG Junior redemption**. The attacker ends **6,965.773429 USDG net profitable** after the Junior-share purchase, both traders' supplied capital, execution fees, bounties and carry. The control performs the same share purchase and round trip without the marked reconciliation: `unassignedAssets` remains 40,000 USDG, Junior has zero withdrawal capacity, and the attacker loses 997.120998 USDG.

## Root cause

`CfdEngineProtocolLens.getHousePoolInputSnapshot` supplies physical custody and terminal price PnL as separate values:

```solidity
snapshot.physicalAssets = poolAssets;
snapshot.netPhysicalAssets = poolAssets;
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
```

`HousePoolAccountingLib.buildReconcileSnapshot` then applies the signed terminal delta to terminal assets/liabilities. This is appropriate for temporary tranche NAV pricing:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}
```

It is sufficient to cap what may safely be treated as distributable at that mark. It is not sufficient evidence that a separate persistent ownership bucket has suffered a realized loss. Nevertheless, `HousePoolReconcilePlanLib.planReconcile` writes that cap into the bucket itself:

```solidity
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

`HousePool._reconcile` persists the lower value:

```solidity
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;
```

Let physical canonical assets be `P`, quarantined assets be `U`, stored Senior plus Junior principal be `E`, and a temporary marked trader liability be `L`. Initially `P = U + E`. If `L > E`, marked distributable equity is below the reserve and reconciliation stores:

```text
U' = P - L < U
```

If the mark later reverses without realized trader price PnL, custody remains approximately `P`, but reconciliation reserves only `U'`. It therefore classifies approximately `P - U' = L` as claimant equity. Senior is restored first and the remainder is credited to Junior. The missing `U - U'` changed ownership even though no payout, assignment or realized price loss consumed it.

The bug is the irreversible ownership write, not terminal-NAV pricing or the temporary effective cap itself.

## Why the documented cap does not resolve the issue

The code comment that `unassignedAssets` is "capped to remaining distributable value" accurately describes the calculation and protects the pool from treating marked liabilities as simultaneously withdrawable reserve. This report does not argue that the protocol should ignore terminal liabilities.

The missing step is reversibility. At the adverse mark in the PoC:

- no attack position closes;
- no realized price PnL or deferred trader claim is created;
- no USDG leaves the HousePool; and
- governance does not assign any part of the reserve.

Despite the absence of a source-changing event, reconciliation commits the lower value to the only persistent `unassignedAssets` ledger. Once the mark reverses, the implementation restores tranche principal against that lower baseline but never restores the reserve classification. A marked cap and a persistent ownership mutation are therefore materially different operations.

## Why the reserve precondition is legitimate

The PoC does not recapitalize a healthy pool and does not require malicious or deviant governance conduct:

1. Senior and Junior are funded through public asynchronous deposits.
2. Two ordinary V3 positions create an actual settled LP loss; the losing close executes first so recovery does not depend on degraded-mode ordering.
3. Public LP settlement records Junior wipeout and Senior impairment.
4. Governance calls the documented immediate recovery action with exactly:

```text
current Senior deficit + 40,000 USDG
```

5. Reconciliation restores Senior exactly and places the remaining 40,000 USDG in `unassignedAssets`.

This is the state the implementation expressly supports. The repository's `EngineRecapitalizationTest.test_RecapitalizePool_RestoresSeniorWithoutMintingShares` likewise restores an impaired Senior tranche and deliberately leaves recapitalization surplus in `unassignedAssets`; its 10,000-USDG fixture demonstrates the same surplus-routing path. The PoC uses 40,000 USDG so that, after reclassification, Junior exceeds the release covenant's required 25%-of-Senior backing and a public redemption can receive funds.

The privileged action only creates the documented reserve. Every operation that reallocates or withdraws it is performed later by untrusted users through public entry points.

## Attack sequence

1. A completed loss/recovery cycle leaves Senior restored and `unassignedAssets = 40,000 USDG`.
2. Junior principal is zero, while the wiped Junior ERC-20 shares still exist. Their full protocol conversion value is zero.
3. The attacker controls the surviving wiped shares. This can be the original wiped holder; the PoC instead transfers them through the public ERC-20 path for 1 USDG to isolate the economics of acquiring a zero-protocol-NAV claim. The accounting exploit does not depend on that particular consideration.
4. Two attacker-controlled accounts each receive 10,000 USDG and open balanced 132,000-unit SHORT and LONG positions at `1.00`. The positions are built in four alternating 33,000-unit chunks, so each public open satisfies the release 40% skew rule.
5. The attacker queues all acquired Junior shares for redemption.
6. At the valid protocol cap mark `2.00`, permissionless `settleLpEpoch` incorporates the live terminal book. The net marked trader liability lowers stored `unassignedAssets` from 40,000 to 8,096.350153 USDG and wipes current tranche principal. No USDG leaves the pool and no trader claim is created at this step. The zero-NAV Junior request is refunded, so no shares are burned.
7. The same LONG and SHORT positions close at their `1.00` entry price. Neither attack trader realizes price PnL.
8. The attacker queues the preserved Junior shares again. Settlement reserves only the surviving 8,096.350153 USDG, restores Senior, and credits the ownership difference plus ordinary order revenue to Junior. The release 80% Senior-share covenant leaves at least 25% of protected Senior as Junior backing and funds only the excess.
9. The attacker redeems that **7,964.297887-USDG excess**. Both trading accounts withdraw 19,002.475542 USDG. Against 20,001 USDG supplied by the attacker, the realized net profit is 6,965.773429 USDG.

## Feasibility

The reserve starts shrinking only when the marked trader liability exceeds the currently claimed Senior-plus-Junior principal. To first order, the required relative mark move is therefore greater than `claimed tranche principal / one-sided position size`. Immediately before the reversible mark, the PoC has 99,001.905670 USDG of claimed principal against a 132,000-unit one-sided position, while the protocol measures 130,932.000338 USDG of terminal trader liability at the 100% cap move. The measured crossing ratio is about **75.61%**, or approximately 76% after execution-cost and terminal-accounting effects. The PoC crosses it with a 100% move from `1.00` to the configured `2.00` cap.

The recovery setup also requires a large move: its separate balanced book opens at `1.00`, closes at `1.80` to wipe Junior and impair Senior, and the mark returns to `1.00` when the attack book opens. The complete tested price sequence is therefore `1.00 -> 1.80 -> 1.00 -> 2.00 -> 1.00`, not merely the reversible `1.00 -> 2.00 -> 1.00` attack phase.

These large moves for the six-currency basket are the report's main feasibility limitation. The `2.00` mark must be observed while the attack book remains open and a permissionless LP epoch is settled. The construction does not assume a price outside the configured range, a forged oracle update, or an altered release parameter.

The Scope page's **Feasibility Limitations** section states:

> “Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it.”

Accordingly, the endpoint requirement can reduce payout or practical likelihood, but it does not change the demonstrated accounting mutation or its Medium impact classification.

## Measured impact

| Quantity | Attack | Control |
| --- | ---: | ---: |
| Initial quarantined reserve | 40,000.000000 | 40,000.000000 |
| Reserve deleted/reassigned | 31,903.649847 | 0 |
| Junior redemption | 7,964.297887 | 0 |
| Trader capital recovered | 19,002.475542 | 19,003.879002 |
| Attacker result after all supplied capital | **+6,965.773429** | **-997.120998** |

The attack/control outcome difference is 7,962.894427 USDG, closely tracking the 7,964.297887-USDG funded Junior exit after the small difference in order costs. The larger 31,903.649847-USDG ownership deletion is not described as wholly extracted: the release 80% covenant retains the rest as Junior backing for protected Senior exposure. The control's ordinary execution revenue remains below that required backing, so its Junior withdrawal capacity is zero.

No attack trade receives favorable execution: both attack positions open and close at `1.00`. The `2.00` observation is used only for the intermediate terminal-NAV reconciliation. The profit comes from the persistent ownership mutation.

Because `unassignedAssets` has no current claimant, this evidence does not establish theft of a named LP's principal or of user-owned "unclaimed yield." The exact demonstrated loss is to protocol accounting and its governance-funded recovery reserve. The applicable in-scope impact is therefore the Medium category **"Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)"**. The fact that this construction is profitable strengthens exploitability but does not turn ownerless value into proven user funds.

## Intended behavior

The repository consistently describes this value as excluded from ordinary LP ownership:

- `IHousePool.unassignedAssets` calls it canonical assets "reserved from ordinary LP withdrawals and deposits."
- `HousePool.assignUnassignedAssets` consumes the bucket while minting matching shares and says this prevents LPs from implicitly capturing previously ownerless value.
- `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` expressly routes recapitalization remainder to `unassignedAssets`: "Any remainder [...] becomes unassigned; recapitalization never accrues to junior principal."
- `ACCOUNTING_SPEC.md` says value without a valid claimant path must remain in the explicit bucket and that a depositor cannot acquire it merely by entering first.
- `PerpWaterfallReferenceInvariant` asserts that recapitalization surplus "remains reserved through later trading revenue."

The initial recapitalization follows that rule and correctly places the 40,000-USDG remainder in `unassignedAssets`. The later reversible-mark sequence nevertheless converts 31,903.649847 USDG of that same recapitalization remainder into Junior principal and immediately makes 7,964.297887 USDG withdrawable under the release covenant. It therefore defeats a written recapitalization-routing rule, not merely an ambiguous interpretation of the cap comment, and does so without calling `assignUnassignedAssets`.

## Distinction from prior audit findings

The closest disclosed item is **MED-26, "Owner can activate trading against ownerless pool assets."** Its root cause is an activation gate: `activateTrading()` allowed the initial false-to-true transition while `unassignedAssets` was nonzero. Its fix rejects that initial activation, while expressly leaving trading continuous if ownerless value arises after activation. MED-26 therefore accepts that ongoing positions can rely on ownerless cash and that an actually realized trader win can consume it.

This report does not reactivate trading, does not claim that trading must be continuously blocked while the reserve exists, and does not dispute consumption by an actually realized payout. The pool is activated before the loss/recovery event. At the adverse mark here, however, no position closes, no trader claim is created and no cash is paid. The distinct root is the persistent assignment:

```solidity
unassignedAssets = min(unassignedAssets, terminalDistributable)
```

under **unrealized** PnL, combined with the absence of any reversal ledger. Even if ongoing trading against ownerless cash is accepted, a temporary mark must not permanently change the source classification of physical USDG after every position returns to its entry price. MED-26's activation guard cannot prevent or repair this write.

The library comment saying that unassigned assets are "capped to remaining distributable value" describes the implemented calculation, but it does not disclose or justify the irreversible result proved here. The PoC ends with the same positions closed at their entry price and no realized price PnL, yet the cap is never restored and the holder of wiped Junior shares withdraws the difference.

No disclosed audit item located during review describes permanent deletion of `unassignedAssets` under a reversible unrealized mark.

## Recommendation

Do not persistently reduce the unassigned ownership ledger from an unrealized terminal-PnL snapshot. Separate:

1. an effective marked reserve used temporarily for NAV, preview and withdrawal gating; and
2. the persistent ownership ledger, which decreases only when a realized physical outflow/loss actually consumes quarantined cash or governance assigns it.

If marked impairment must be persisted, record it separately and restore `unassignedAssets` on reversal before classifying recovered value as Senior/Junior revenue.

Add a stateful invariant for `mark A -> mark B -> mark A` with unchanged position entry/exit and no realized price PnL: after fees are isolated, the ownership split among `unassignedAssets`, Senior and Junior must return to its starting state.

## References

- Destructive cap: `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-85`
- Persistent write: `packages/perps/src/HousePool.sol:1555-1598`
- Terminal snapshot construction: `packages/perps/src/libraries/HousePoolAccountingLib.sol:57-77`
- Engine input lens: `packages/perps/src/CfdEngineProtocolLens.sol:49-89`
- Exact terminal curve: `packages/perps/src/TerminalNavBookV2.sol:320-350`
- Explicit assignment semantics: `packages/perps/src/HousePool.sol:653-703`
- Reserved-bucket interface: `packages/perps/src/interfaces/IHousePool.sol:369-371`
- Recapitalization never accrues to Junior: `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-107`
- Accounting specification: `packages/perps/ACCOUNTING_SPEC.md:308-357, 628-652`
- Documented recapitalization-surplus test: `packages/perps/test/perps/spec/admin/EngineRecapitalization.t.sol:33-61`
- Existing reserve invariant: `packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol:309-320`
- Competition feasibility rule: [Plether Scope page](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept

# Proof of Concept

Save the following file as `packages/perps/test/perps/regression/UnassignedAssetMarkReversal.t.sol` in the frozen repository. The complete runnable source is included below so the submission is self-contained.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";

/// @notice A reversible terminal mark permanently reallocates a documented recapitalization surplus.
/// @dev The precondition is reached through public V3 trading followed by the owner's documented recovery action.
///      Only Pyth and USDG are mocked. No pool balance or in-scope storage is forged, and no in-scope contract is
///      impersonated. The attacker begins only after recovery has restored Senior and quarantined the surplus.
contract UnassignedAssetMarkReversalTest is HousePoolAsyncTestBase {

    address internal constant JUNIOR_SEED_RECEIVER = address(0x5100);
    address internal constant SENIOR_SEED_RECEIVER = address(0x5200);
    address internal constant SENIOR_LP = address(0xA11CE);
    address internal constant JUNIOR_LP = address(0xB0B);
    address internal constant ATTACK_JUNIOR = address(0xA770);
    address internal constant RECOVERY_WINNER = address(0xC001);
    address internal constant RECOVERY_LOSER = address(0xC002);
    address internal constant ATTACK_SHORT = address(0xD001);
    address internal constant ATTACK_LONG = address(0xD002);
    address internal constant PUBLIC_KEEPER = address(0xBEEF);

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant RECOVERY_LOSS_PRICE = 18e7;
    uint256 internal constant REVERSIBLE_MARK = 2e8;

    uint256 internal constant SENIOR_LP_DEPOSIT = 99_000e6;
    uint256 internal constant JUNIOR_LP_DEPOSIT = 99_000e6;
    uint256 internal constant QUARANTINED_SURPLUS = 40_000e6;
    uint256 internal constant JUNIOR_SHARE_PURCHASE = 1e6;

    uint256 internal constant RECOVERY_SIZE = 190_000e18;
    uint256 internal constant RECOVERY_CHUNK_SIZE = 47_500e18;
    uint256 internal constant RECOVERY_CHUNK_MARGIN = 2500e6;
    uint256 internal constant RECOVERY_TRADER_FUNDING = 30_000e6;

    uint256 internal constant ATTACK_CHUNK_SIZE = 33_000e18;
    uint256 internal constant ATTACK_TOTAL_SIZE = 132_000e18;
    uint256 internal constant ATTACK_CHUNK_MARGIN = 300e6;
    uint256 internal constant ATTACK_TRADER_FUNDING = 10_000e6;

    struct AttackResult {
        uint256 deletedUnassigned;
        uint256 juniorPayout;
        uint256 traderRecovery;
        uint256 netProfit;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return JUNIOR_SEED_RECEIVER;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_SEED_RECEIVER;
    }

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

        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);

        // The two governance timelocks above end during the configured weekend closure.
        // Move to the next ordinary trading day before using public LP and order entry points.
        vm.warp(vm.getBlockTimestamp() + 1 days);
    }

    function _activateDeposit(
        TrancheVault vault,
        address lp,
        uint256 assets,
        uint256 markPrice
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();

        uint256 activationTime = vault.depositEpochStart(requestId);
        if (block.timestamp < activationTime) {
            vm.warp(activationTime);
        }
        _settleEpoch(markPrice);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, assets, "deposit basis");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    function _executeOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 executionPrice
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()) + uint256(uint160(account)));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = executionPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
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
        vm.prank(PUBLIC_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(executionPrice));
        assertEq(result.orderId, orderId, "executed order id");
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order status");
    }

    function _settleEpoch(
        uint256 price
    ) internal {
        vm.prank(PUBLIC_KEEPER);
        router.settleLpEpoch(_mockPythUpdateData(price));
    }

    function _settleThroughJuniorRefund(
        uint256 price
    ) internal {
        _finishAsyncCooldown(juniorVault, JUNIOR_LP);
        uint256 shares = juniorVault.balanceOf(JUNIOR_LP);
        uint256 requestId = _requestAsyncRedeem(juniorVault, JUNIOR_LP, shares);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
        _settleEpoch(price);
        assertEq(juniorVault.refundableRedeemRequest(requestId, JUNIOR_LP), shares, "Junior must remain wiped");
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(requestId, JUNIOR_LP, JUNIOR_LP);
    }

    /// @dev Creates the same class of state exercised by EngineRecapitalizationTest: a real Senior impairment is
    ///      restored and an intentional 40,000-USDG excess remains quarantined. The impairment is produced by two
    ///      public, fully settled positions rather than by burning pool tokens.
    function _establishDocumentedRecoveryState() internal {
        _activateDeposit(juniorVault, JUNIOR_LP, JUNIOR_LP_DEPOSIT, ENTRY_PRICE);
        _activateDeposit(seniorVault, SENIOR_LP, SENIOR_LP_DEPOSIT, ENTRY_PRICE);
        assertApproxEqAbs(pool.seniorPrincipal(), SENIOR_LP_DEPOSIT + 1e6, 2e6, "Senior setup");
        assertApproxEqAbs(pool.juniorPrincipal(), JUNIOR_LP_DEPOSIT + 1e6, 2e6, "Junior setup");

        _fundTrader(RECOVERY_WINNER, RECOVERY_TRADER_FUNDING);
        _fundTrader(RECOVERY_LOSER, RECOVERY_TRADER_FUNDING);
        for (uint256 i; i < 4; ++i) {
            _executeOrder(
                RECOVERY_WINNER, CfdTypes.Side.SHORT, RECOVERY_CHUNK_SIZE, RECOVERY_CHUNK_MARGIN, false, ENTRY_PRICE
            );
            _executeOrder(
                RECOVERY_LOSER, CfdTypes.Side.LONG, RECOVERY_CHUNK_SIZE, RECOVERY_CHUNK_MARGIN, false, ENTRY_PRICE
            );
        }
        _executeOrder(RECOVERY_LOSER, CfdTypes.Side.LONG, RECOVERY_SIZE, 0, true, RECOVERY_LOSS_PRICE);
        _executeOrder(RECOVERY_WINNER, CfdTypes.Side.SHORT, RECOVERY_SIZE, 0, true, RECOVERY_LOSS_PRICE);

        _settleThroughJuniorRefund(RECOVERY_LOSS_PRICE);
        uint256 highWaterMark = pool.seniorHighWaterMark();
        uint256 impairedSenior = pool.seniorPrincipal();
        assertEq(pool.juniorPrincipal(), 0, "real trading loss must wipe Junior");
        assertLt(impairedSenior, highWaterMark, "real trading loss must impair Senior");

        uint256 seniorDeficit = highWaterMark - impairedSenior;
        uint256 recapitalization = seniorDeficit + QUARANTINED_SURPLUS;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        _settleThroughJuniorRefund(RECOVERY_LOSS_PRICE);

        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "Senior recovery must be exact");
        assertEq(pool.unassignedAssets(), QUARANTINED_SURPLUS, "documented recap surplus must be quarantined");
    }

    function _acquireWipedJuniorShares() internal {
        uint256 shares = juniorVault.balanceOf(JUNIOR_LP);
        assertGt(shares, 0, "seller must retain wiped Junior shares");
        assertEq(juniorVault.convertToAssets(shares), 0, "shares must have zero protocol NAV before purchase");

        settlementToken.mint(ATTACK_JUNIOR, JUNIOR_SHARE_PURCHASE);
        vm.prank(ATTACK_JUNIOR);
        settlementToken.transfer(JUNIOR_LP, JUNIOR_SHARE_PURCHASE);
        _finishAsyncCooldown(juniorVault, JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.transfer(ATTACK_JUNIOR, shares);
        assertEq(juniorVault.balanceOf(ATTACK_JUNIOR), shares, "public ERC20 share transfer failed");
    }

    function _openBalancedAttackBook() internal {
        _fundTrader(ATTACK_SHORT, ATTACK_TRADER_FUNDING);
        _fundTrader(ATTACK_LONG, ATTACK_TRADER_FUNDING);
        for (uint256 i; i < 4; ++i) {
            _executeOrder(ATTACK_SHORT, CfdTypes.Side.SHORT, ATTACK_CHUNK_SIZE, ATTACK_CHUNK_MARGIN, false, ENTRY_PRICE);
            _executeOrder(ATTACK_LONG, CfdTypes.Side.LONG, ATTACK_CHUNK_SIZE, ATTACK_CHUNK_MARGIN, false, ENTRY_PRICE);
        }
    }

    function _closeBalancedAttackBook() internal {
        _executeOrder(ATTACK_SHORT, CfdTypes.Side.SHORT, ATTACK_TOTAL_SIZE, 0, true, ENTRY_PRICE);
        _executeOrder(ATTACK_LONG, CfdTypes.Side.LONG, ATTACK_TOTAL_SIZE, 0, true, ENTRY_PRICE);

        (uint256 shortSize,,,,,,) = engine.positions(ATTACK_SHORT);
        (uint256 longSize,,,,,,) = engine.positions(ATTACK_LONG);
        assertEq(shortSize, 0, "SHORT must finish flat");
        assertEq(longSize, 0, "LONG must finish flat");
    }

    function _withdrawAttackTraderBalances() internal returns (uint256 recovered) {
        uint256 shortAssets = clearinghouse.balanceAssets(ATTACK_SHORT);
        uint256 longAssets = clearinghouse.balanceAssets(ATTACK_LONG);
        vm.prank(ATTACK_SHORT);
        clearinghouse.withdraw(ATTACK_SHORT, shortAssets);
        vm.prank(ATTACK_LONG);
        clearinghouse.withdraw(ATTACK_LONG, longAssets);
        recovered = settlementToken.balanceOf(ATTACK_SHORT) + settlementToken.balanceOf(ATTACK_LONG);
    }

    function _redeemAllJunior(
        uint256 price
    ) internal returns (uint256 assets) {
        _finishAsyncCooldown(juniorVault, ATTACK_JUNIOR);
        uint256 shares = juniorVault.balanceOf(ATTACK_JUNIOR);
        uint256 requestId = _requestAsyncRedeem(juniorVault, ATTACK_JUNIOR, shares);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
        _settleEpoch(price);
        (, assets) = _claimAsyncRedeem(juniorVault, requestId, ATTACK_JUNIOR);
    }

    function test_ReversibleMarkPermanentlyTransfersRecoveredReserveToWipedJuniorShares() public {
        _establishDocumentedRecoveryState();
        _acquireWipedJuniorShares();
        uint256 unassignedBefore = pool.unassignedAssets();
        assertEq(unassignedBefore, QUARANTINED_SURPLUS);
        assertEq(pool.pendingRecapitalizationAssets(), 0, "recovery must be fully reconciled before the attack");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no pending revenue may explain the later assignment");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "recovery state must have no outstanding trader claim");

        _openBalancedAttackBook();
        emit log_named_decimal_uint(
            "claimed tranche principal before reversible mark", pool.seniorPrincipal() + pool.juniorPrincipal(), 6
        );

        // Queue the Junior exit before the temporary mark. At the adverse epoch its zero NAV is refunded, which
        // commits reconciliation while preserving every attacker share.
        _finishAsyncCooldown(juniorVault, ATTACK_JUNIOR);
        uint256 shares = juniorVault.balanceOf(ATTACK_JUNIOR);
        uint256 adverseRequest = _requestAsyncRedeem(juniorVault, ATTACK_JUNIOR, shares);
        uint256 maturity = pool.lpEpochStart(adverseRequest);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
        uint256 poolCashBeforeMark = settlementToken.balanceOf(address(pool));
        _settleEpoch(REVERSIBLE_MARK);

        int256 terminalLpPriceDeltaAssets = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
        assertLt(terminalLpPriceDeltaAssets, 0, "temporary mark must favor traders");
        emit log_named_decimal_uint("terminal marked trader liability at cap", uint256(-terminalLpPriceDeltaAssets), 6);
        assertEq(
            settlementToken.balanceOf(address(pool)),
            poolCashBeforeMark,
            "temporary mark must not move canonical pool USDG"
        );
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "temporary mark must not create a realized trader claim");
        uint256 unassignedAfterMark = pool.unassignedAssets();
        assertLt(unassignedAfterMark, unassignedBefore, "temporary mark must delete reserved ownership");
        assertEq(pool.seniorPrincipal(), 0, "marked liability must wipe current Senior principal");
        assertEq(juniorVault.refundableRedeemRequest(adverseRequest, ATTACK_JUNIOR), shares, "shares must survive");
        vm.prank(ATTACK_JUNIOR);
        juniorVault.claimRedeemRefund(adverseRequest, ATTACK_JUNIOR, ATTACK_JUNIOR);

        // The same positions close at their entry, so neither attack trader realizes price PnL. The next settlement
        // restores Senior against the smaller U ledger and credits the deleted reserve to the acquired Junior shares.
        _closeBalancedAttackBook();
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "same-price closes must leave no trader claim");

        (uint256 previewSenior, uint256 previewJunior,, uint256 previewJuniorWithdraw) = pool.getPendingTrancheState();
        assertEq(previewSenior, pool.seniorHighWaterMark(), "reversal preview must restore Senior exactly");
        assertGt(previewJunior, previewSenior / 4, "reclassified Junior must exceed the release 80% covenant reserve");
        assertGt(previewJuniorWithdraw, 0, "release 80% covenant must permit a real Junior payout");

        uint256 juniorPayout = _redeemAllJunior(ENTRY_PRICE);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "reversal settlement must restore Senior exactly");
        assertApproxEqAbs(
            juniorPayout, previewJuniorWithdraw, 2, "settlement must fund the previewed extractable excess"
        );
        assertGe(
            pool.juniorPrincipal() * 4,
            pool.seniorHighWaterMark(),
            "partial redemption must retain the release covenant reserve"
        );
        uint256 traderRecovery = _withdrawAttackTraderBalances();

        assertEq(pool.unassignedAssets(), unassignedAfterMark, "reversal must not restore deleted ownership");

        uint256 attackerSupplied = JUNIOR_SHARE_PURCHASE + 2 * ATTACK_TRADER_FUNDING;
        uint256 attackerRecovered = juniorPayout + traderRecovery;
        assertLt(
            traderRecovery,
            2 * ATTACK_TRADER_FUNDING,
            "same-price trading legs must lose value to fees, bounty and carry"
        );
        assertGt(juniorPayout, 0, "deleted reserve must fund a real Junior redemption");
        assertLt(juniorPayout, unassignedBefore - unassignedAfterMark, "80% covenant must retain part of the value");
        assertGt(attackerRecovered, attackerSupplied, "reserve transfer must exceed every attacker-supplied asset");

        AttackResult memory result = AttackResult({
            deletedUnassigned: unassignedBefore - unassignedAfterMark,
            juniorPayout: juniorPayout,
            traderRecovery: traderRecovery,
            netProfit: attackerRecovered - attackerSupplied
        });
        emit log_named_decimal_uint("unassigned ownership deleted", result.deletedUnassigned, 6);
        emit log_named_decimal_uint("Junior redemption paid", result.juniorPayout, 6);
        emit log_named_decimal_uint("attack trader capital recovered", result.traderRecovery, 6);
        emit log_named_decimal_uint("attacker net profit", result.netProfit, 6);
    }

    function test_Control_RoundTripWithoutMarkedReconciliationCannotCaptureReserve() public {
        _establishDocumentedRecoveryState();
        _acquireWipedJuniorShares();
        _openBalancedAttackBook();
        _closeBalancedAttackBook();

        (uint256 previewSenior, uint256 previewJunior,, uint256 previewJuniorWithdraw) = pool.getPendingTrancheState();
        assertEq(previewJuniorWithdraw, 0, "control must not make Junior withdrawable under the 80% covenant");
        assertLt(previewJunior, previewSenior / 4, "control Junior value must remain below the covenant reserve");

        uint256 juniorPayout;
        uint256 traderRecovery = _withdrawAttackTraderBalances();
        uint256 attackerSupplied = JUNIOR_SHARE_PURCHASE + 2 * ATTACK_TRADER_FUNDING;

        assertEq(pool.unassignedAssets(), QUARANTINED_SURPLUS, "ordinary round trip must preserve U");
        assertLt(juniorPayout + traderRecovery, attackerSupplied, "control must only pay execution costs");
        emit log_named_decimal_uint("control Junior payout", juniorPayout, 6);
        emit log_named_decimal_uint("control attacker cost", attackerSupplied - juniorPayout - traderRecovery, 6);
    }

}
```

It uses:

- the release risk parameters, including the 40% skew limit;
- the release 40,000,000-USDG/80% Senior-capacity settings and 1-USDG seed positions;
- the release 1,000-USDG minimum open notional and 2,500-bps adverse-confidence multiplier;
- public V3 `commitOrder` and permissionless `executeOrder` for every open and close;
- public permissionless `settleLpEpoch` for every reconciliation;
- public asynchronous deposit, redeem, refund and claim paths;
- a public ERC-20 transfer of zero-NAV Junior shares after the buyer pays the seller 1 USDG (an original wiped holder could execute the same accounting sequence without this transfer);
- the documented owner `recapitalizePool` action only after an actual Senior impairment, for the exact deficit plus a 40,000-USDG surplus routed through the same quarantine path as the repository's 10,000-USDG test case;
- only the repository's Pyth and USDG mocks;
- no `vm.store`, pool-token burn, in-scope-contract impersonation, forged in-scope balance, legacy Router entrypoint or state rollback.

Run from the frozen repository:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/UnassignedAssetMarkReversal.t.sol -vv
```

Observed result:

```text
Ran 2 tests for test/perps/regression/UnassignedAssetMarkReversal.t.sol:UnassignedAssetMarkReversalTest
[PASS] test_Control_RoundTripWithoutMarkedReconciliationCannotCaptureReserve()
Logs:
  control Junior payout: 0.000000
  control attacker cost: 997.120998

[PASS] test_ReversibleMarkPermanentlyTransfersRecoveredReserveToWipedJuniorShares()
Logs:
  claimed tranche principal before reversible mark: 99001.905670
  terminal marked trader liability at cap: 130932.000338
  unassigned ownership deleted: 31903.649847
  Junior redemption paid: 7964.297887
  attack trader capital recovered: 19002.475542
  attacker net profit: 6965.773429

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

All logged USDG values use six decimals.

