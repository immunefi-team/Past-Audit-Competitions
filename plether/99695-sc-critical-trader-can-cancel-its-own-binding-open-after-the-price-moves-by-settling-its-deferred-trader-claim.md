# \#99695 [SC-Critical] Trader can cancel its own binding open after the price moves by settling its deferred trader claim

**Submitted on Nov 14th 2026 at 20:34:27 UTC by @satyaorz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99695
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Plether's delayed orders are documented as **binding**: "Users cannot cancel queued orders once committed"
(SECURITY.md:366). The protocol states that binding semantics and the absence of a user cancellation path "prevent
traders from turning queued intents into free options" (SECURITY.md:658).

A trader that is flat and holds a deferred **trader claim** can still cancel its own committed open. Open admission
counts the same-account claim as price-risk collateral. `settleTraderClaim`, which only the account itself can call,
has no pending-order guard, and for a flat account it converts the claim into free settlement, which admission does
not count. If the trader settles the claim after committing, execution re-plans the open, which fails
`INSUFFICIENT_INITIAL_MARGIN` and is rejected terminally.

The trader can wait until the fill tick and the later price are both public, then keep the order if the price moved
in its favour or cancel it if it moved against it. (Plether's basket price is inverted relative to the dollar: a
LONG gains when the price *falls*, so in the PoC a rise from P1 = 1.00 to P2 = 1.01 is adverse for the LONG.) Under production parameters the PoC measures this option at
**3,241.20 USDG** for one 300,000 USDG order. Relative to the binding outcome the LPs forgo a **3,121.20 USDG** gain;
no deposited value is taken from them.

> **Precondition.** The attacker is a flat account holding a deferred trader claim, a documented protocol state
> (ACCOUNTING_SPEC.md:35, :50 define it; :1092 counts it as admission collateral). In the PoC the claim is created
> organically by a winning close whose payout exceeded free pool cash. It stays deferred after the owner performs the documented recovery in SECURITY.md
> :1499-1510 (`recapitalizePool`, then `clearDegradedMode` once `E >= L`). At the moment of the attack the pool is
> recapitalized and solvent, the engine is not degraded and opens are enabled. No withdrawal, payout or redemption is
> blocked or short of liquidity. The impact is the order-cancellation primitive, not a liquidity shortfall.

## Vulnerability Details

**1. Open admission counts the same-account claim as collateral.** The initial-margin check builds equity from the
new position margin plus the account's trader claim. Free settlement is not included:

```solidity
// CfdEnginePlanLib.sol:664-667  (_buildPostOpenRiskState)
uint256 reachableCollateralAssets = delta.positionMarginAfterOpen + snap.traderClaimBalanceForAccount;

// CfdEnginePlanLib.sol:431-436  (planOpen)
if (postOpenRiskState.liquidatable
        || postOpenRiskState.equityAssets <= int256(openState.initialMarginRequirementAssets)) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN;
```

The same rule is specified in ACCOUNTING_SPEC:1092. A flat claimant can therefore commit an open whose
`marginDelta` only clears initial margin once the claim is added.

**2. The claim can be removed while the order is pending.** `settleTraderClaim` checks only `msg.sender == account`
and that pool cash covers all outstanding trader claims (settlement is all-or-nothing). It does not check for pending
orders (CfdEngine.sol:559-590). For a flat account the payout becomes **free settlement** (CfdEngine.sol:233-243,
`_payTraderClaim` → `clearinghouse.settleAssets`), which is not admission collateral. For an account with an open position the payout would instead be credited to pledge and stay
collateral; that is why the account must be flat.

The spec already recognizes this hazard, but guards only one of the two places that rely on the claim.
ACCOUNTING_SPEC:771-773: for a live position, settled claim value is credited to pledge and "must not become
withdrawable free settlement while its value was already included in the marked collectible cap". Open admission
relies on the same claim value (ACCOUNTING_SPEC:1092; CfdEnginePlanLib:667), yet a pending open has no equivalent
guard. Settled claim value becomes withdrawable free settlement even though a committed open's admission was granted
against it.

**3. Execution re-plans and rejects terminally.** At execution the planner reads the account's current claim, now
zero. The open fails `INSUFFICIENT_INITIAL_MARGIN`, which `getExecutionFailurePolicyCategory` classifies as
`UserInvalid` (CfdEnginePlanLib.sol:112-129). The Router settles it as `PlannerRejected` and pays the opening bounty
to the executor (`OrderRouter._settleTerminalBounty`, OrderRouter.sol:314-337). If the trader is its own
executor, the bounty is released back to it (`bountyRecipient == order.account → releaseReservedExecutionBountyToSource`).

**4. Why this is more than the built-in slippage limit.** A tight `targetPrice` can make an open fail when the first
post-commit tick is unfavourable, but that condition is fixed at commit time. Here the trader decides *after* both the
fill tick P1 and the later price P2 are public. It keeps the order when the market has moved in its favour relative
to P1 and cancels it when the market has moved against it. This is the free option that SECURITY.md:658 says binding
semantics prevent.

**5. Mitigations that do not apply.** The order settlement floor (`bounds.minPostSettlementBalanceAssets`) is enforced
through the withdraw guard (CfdEngineSettlementSidecar.sol:260-272). `settleTraderClaim` is a separate entrypoint, and
it only *increases* the settlement balance, so even a maximal floor is satisfied.

## Impact Details

- **Broken guarantee (the promised return not delivered):** a committed open is no longer binding for a flat
  claimant. It becomes a free option over the keeper's execution latency. LPs do not lose deposited value: their net
  assets are unchanged in the attack branch (measured 0). They forgo the gain the binding order would have delivered.
- **Measured value (production parameters, 300,000 USDG LONG, 1% adverse move):** this is a control-vs-attack
  counterfactual measured from one snapshot, not a realized transfer inside the attack branch.
  - Control (order kept): trader **−3,241.40 USDG**, LP net assets **+3,121.20 USDG**.
  - Attack (claim settled before execution): trader **−0.20 USDG** (opening bounty), LP net assets **0**.
  - Trader option value **3,241.20 USDG**; LP gain forgone **3,121.20 USDG**.
  - Self-executed: the cancellation costs **zero internal USDG**; gas and Pyth update fees are still paid in ETH.
- **Limits:**
  - One cancellation per claim, because using it consumes the claim. Every open the claimant commits before then
    carries the option.
  - The trader must size `marginDelta` inside the claim-dependent window. The PoC finds 600 USDG for 300,000 USDG
    notional.
  - The option lasts until the first keeper executes the order. Keepers are permissionless and untrusted, so when the
    attacker is the only executor it lasts up to the order's execution window.
  - The prerequisite claim originates from an earlier deferred payout (see Precondition); this affects likelihood,
    not impact.

Severity is **Low** ("Contract fails to deliver promised returns, but doesn't lose value"). A documented protocol
guarantee is broken by an untrusted trader. No deposited value is lost; the forgone return is the option value on one
order per claim, and it is not repeatable at will.

## Program rules compliance

- **Trusted roles:** no role is malicious, compromised or acting outside its documented procedures. The only
  privileged calls are the owner's documented recovery (SECURITY.md:157, :1499-1510;
  EMERGENCY_RESPONSE_GUIDE.md:47). The program states: "In scope: a legitimate, documented action by a trusted role
  that triggers a bug harming users or the protocol." The attacker is an ordinary, untrusted trader.
- **Liquidity:** creating the deferred claim requires an earlier pool-cash shortfall. The exploited cancellation
  occurs only after documented recovery, when the pool is recapitalized and solvent, degraded mode is cleared, and
  opens are enabled. The claimed impact is the post-recovery binding-order violation, not the earlier liquidity
  shortfall.
- **Feasibility:** the claim prerequisite affects likelihood, not impact. Per the program, "Feasibility limitations do
  not change the impact or severity of a finding, only the amount paid for it."
- **PoC:** runs in the repository's Foundry test suite at the in-scope commit without modifying in-scope contracts.
  Mocks replace only Pyth price updates and USDG, as the program allows. It uses the release parameters from
  `packages/perps/DEPLOYMENT.md`.

## Relation to known issues

- **Pre-audit SC-M4 "Risk Free Option in 60Sec Oracle Window"** (plether-pre-audit-report.pdf p.29). Different
  prerequisites and a different failure path:

  - *Orders:* SC-M4 needs two sequenced orders (BULL/BEAR); this report needs one order.
  - *Price:* SC-M4 relies on keeper-selected ticks within a 60 s window; here the order fills at its unique first
    post-commit tick (historical unique parse).
  - *Cancel trigger:* SC-M4 uses an attacker LP deposit/withdraw that changes pool depth; here it is the trader's own
    same-account claim, reclassified by `settleTraderClaim`.
  - *Failure path:* SC-M4 is `SOLVENCY_EXCEEDED` → `ProtocolStateInvalidated` → RefundUser; here it is
    `INSUFFICIENT_INITIAL_MARGIN` → `UserInvalid` → `PlannerRejected`.

  None of SC-M4's prerequisites (a second order, keeper tick choice, LP deposit/withdraw) is involved here.
- **AI scan: "Deferred trader claims inflate open skew capacity and reduce VPI"** (scan-plether-fi-plether-core-d4541b49.md:206).
  That issue concerns aggregate claims not being deducted from the depth used by the skew cap and VPI, which affects
  a *second* account's open. This report involves no depth, skew or VPI effect. It concerns a *same-account* claim
  reclassified while its own order is pending, and the PoC rejection is `INSUFFICIENT_INITIAL_MARGIN`.
- **SECURITY.md "Router and keeper limitations"** (listed known issue): "known typed planner/policy failures … are
  terminal for the affected order". This report does not dispute that such failures are terminal. It shows that a
  trader can deliberately *induce* one on its own already-committed order, after the price is public, by
  reclassifying its own collateral. That contradicts the first bullet of the same section: "users cannot manually
  cancel queued intents".
- **A separately reported issue on the same binding-intent guarantee** (a skew-reducing open whose negative-VPI
  reserve is funded from withdrawable free settlement, cancelled via `MarginClearinghouse.withdraw`) has a different
  root cause. The collateral class differs: free settlement backing a VPI reserve there, versus a same-account trader
  claim counted as price-risk collateral here. The entrypoint also differs: `withdraw` there, `settleTraderClaim`
  here. Guarding `withdraw` or reserving VPI backing at commit would not stop this path, because `settleTraderClaim`
  only increases the settlement balance and passes every withdraw or settlement-floor check. The two need
  independent fixes.
- **Pyth/Arbitrum clock-alignment limitation** (SECURITY.md:701, HIGH-1). That limitation concerns a fill tick
  that can already be known at submission. Here the order is committed *before* P1 exists and fills at the intended
  unique first post-commit tick. The trader neither chooses nor knows the fill tick in advance; it decides after the
  fill price is public.

## Recommended mitigation

Any one of:
- Revert `settleTraderClaim` for a flat account while it has pending open orders.
- At commit, reserve the claim amount that admission relied on, so that later settlement cannot reclassify it.
- Exclude the claim from open admission collateral for flat accounts. Price-loss netting at close and liquidation
  can keep using it.

## References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:664-667`: claim counted in open admission collateral.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:431-436` (assignment at :435): `INSUFFICIENT_INITIAL_MARGIN`.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:112-129`: execution failure classification.
- `packages/perps/src/CfdEngine.sol:559-590`: `settleTraderClaim` (self-only, no pending-order guard).
- `packages/perps/src/CfdEngine.sol:233-243`: flat-account payout routed to free settlement.
- `packages/perps/src/OrderRouter.sol:314-337` (`_settleTerminalBounty`): bounty paid, or released to the account on self-execution.
- `packages/perps/SECURITY.md:366`, `:658`: binding-intent / no-free-option guarantee. `ACCOUNTING_SPEC.md:1092`.


## Proof of Concept

**Setup:** on a clean checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, save the **Source** below as
`packages/perps/test/perps/ClaimSettlementCancelsBindingOpen.t.sol`. It is a new test file and modifies nothing else.
It extends the repo's own `BasePerpTest` and uses the **production risk parameters** from DEPLOYMENT.md
(`vpiFactor = 0.01e18`, maintenance 0.1%, initial margin 0.2%). Only Pyth and USDG are mocked. There are no storage
writes, no source changes and no impersonation of protocol contracts. `vm.prank` is used only to act as the
attacker (trader), the counterparty, a keeper, and the owner for its documented recovery calls (`recapitalizePool`,
`clearDegradedMode`).

**Steps:**
1. **Deferred claim, organically.** Matched LONG/SHORT open interest is built through `commitOrder`/`executeOrder`.
   The SHORT wins at the cap and closes, draining pool cash. The LONG (the attacker) then wins and closes. Its payout
   exceeds free cash, so it is deferred as a trader claim and the engine degrades. The extreme price path (2.00, then
   0.04) only reaches that state quickly in a test. Any winning close whose payout exceeds free pool cash creates a
   deferred claim (`pricePayoutCreatesClaim`, CfdEnginePlanLib.sol:962), and the path plays no part in the exploit
   itself.
2. **Honest recovery.** The owner calls `recapitalizePool` and `clearDegradedMode`. The engine is now solvent and not
   degraded, and the claim is still deferred.
3. **Claim-dependent margin.** A snapshot scan finds the smallest `marginDelta` that is admitted with the claim and
   rejected without it: 600 USDG for 3,000 lots. `test_claimIsTheAdmissionCollateral` shows that without the claim the
   identical commit reverts with `OrderRouter__PredictableOpenInvalid(6)`.
4. **Commit, then the price becomes public.** The attacker commits the binding open. The unique post-commit tick
   P1 = 1.00 is published, then P2 = 1.01 (adverse for the LONG; see the direction note above) is installed on-chain
   as the engine mark. The test asserts this.
5. **Control vs attack** (from one snapshot). Control: the keeper executes P1, the open fills at P1 and is closed at
   P2. Attack: the attacker calls `settleTraderClaim`, then the keeper executes the same P1 payload, and the open is
   rejected terminally. Balances are measured in both branches.
6. `test_selfExecutedCancellationCostsNoUsdg`: when the attacker executes its own order, the USDG cost is exactly 0.

**Run** (from the repo root, after saving the file):

```
forge test --root packages/perps --match-path 'test/perps/ClaimSettlementCancelsBindingOpen.t.sol' -vv
```

**Output:**

```
Ran 3 tests for test/perps/ClaimSettlementCancelsBindingOpen.t.sol:ClaimSettlementCancelsBindingOpenTest
[PASS] test_claimIsTheAdmissionCollateral()
[PASS] test_claimSettlementCancelsBindingOpenAfterPriceIsPublic()
  claim-dependent margin (USDG 6dp): 600000000
  control: trader change (USDG 6dp): -3241400051
  control: LP net assets change (USDG 6dp): 3121200051
  attack:  trader change (USDG 6dp): -200000
  attack:  LP net assets change (USDG 6dp): 0
  counterfactual: trader option value (USDG 6dp): 3241200051
  counterfactual: LP gain forgone (USDG 6dp): 3121200051
[PASS] test_selfExecutedCancellationCostsNoUsdg()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

**Source:**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title PoC: a flat trader cancels its own binding open by settling its deferred trader claim
/// @notice Open admission counts the same-account trader claim as price-risk collateral
///         (CfdEnginePlanLib._buildPostOpenRiskState). `settleTraderClaim` is self-only and has no pending-order guard;
///         for a flat account it converts the claim into free settlement, which admission does not count. The
///         committed open is re-planned at execution, fails INSUFFICIENT_INITIAL_MARGIN and is terminally rejected.
/// @dev Production risk parameters (DEPLOYMENT.md). Only Pyth and USDG are mocked. No storage writes and no
///      impersonation of protocol contracts; `vm.prank` acts only as the trader, counterparty, a keeper, and the owner
///      for its documented recovery calls (`recapitalizePool`, `clearDegradedMode`).
contract ClaimSettlementCancelsBindingOpenTest is BasePerpTest {
    address internal constant TRADER = address(0xA11CE); // attacker: ends flat holding a deferred claim
    address internal constant COUNTERPARTY = address(0xB0B);
    address internal constant KEEPER = address(0xBEEF);
    uint256 internal constant LOTS = 3000; // 300,000 USDG notional at 1.0

    // ------------------------------------------------------------------ production parameters (DEPLOYMENT.md)
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
        _fundJunior(address(0xCA201), 500_000e6);
        _fundSenior(address(0xDA44), 300_000e6);
        _fundTrader(TRADER, 2_000_000e6);
        _fundTrader(COUNTERPARTY, 2_000_000e6);
        vm.deal(KEEPER, 100 ether);
        _liveMark(1e8);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Main PoC. Order of events: commit -> unique post-commit tick P1 published -> later tick P2 installed
    ///         on-chain as the engine mark -> trader decides -> keeper executes the same historical P1 payload.
    ///         Control and attack branches start from one snapshot; outcomes are measured from balances.
    function test_claimSettlementCancelsBindingOpenAfterPriceIsPublic() public {
        _deferredClaimThenOwnerRecovery();
        uint256 margin = _claimDependentMargin();

        uint64 orderId = _commitOpen(TRADER, CfdTypes.Side.LONG, LOTS, margin);
        uint256 p1 = 1e8;
        bytes[] memory p1Payload = _publishHistoricalTick(p1); // the order's unique first post-commit tick
        uint256 p2 = 1.01e8; // 1% adverse for an inverted LONG (LONG profits when the price falls)
        vm.warp(vm.getBlockTimestamp() + 1);
        _liveMark(p2);
        assertEq(engine.lastMarkPrice(), p2, "P2 is public on-chain before execution");

        uint256 trader0 = _traderWealth();
        int256 lp0 = _lpNetAssets();

        // CONTROL: the order is kept -> fills at P1 -> closed at the already-public P2.
        uint256 snap = vm.snapshotState();
        vm.prank(KEEPER);
        router.executeOrder(orderId, p1Payload);
        (uint256 sizeC,, uint256 entryC,,,,) = engine.positions(TRADER);
        assertEq(sizeC, LOTS * 1e20, "control: binding open filled");
        assertEq(entryC, p1, "control: filled at P1");
        _closeAll(TRADER, p2);
        int256 controlTrader = int256(_traderWealth()) - int256(trader0);
        int256 controlLp = _lpNetAssets() - lp0;
        vm.revertToState(snap);

        // ATTACK: the trader saw P2, settles its claim, the keeper executes the same P1 payload.
        vm.prank(TRADER);
        engine.settleTraderClaim(TRADER);
        vm.prank(KEEPER);
        router.executeOrder(orderId, p1Payload);
        (uint256 sizeA,,,,,,) = engine.positions(TRADER);
        assertEq(sizeA, 0, "attack: binding open cancelled after P2 became public");
        assertEq(router.pendingOrderCounts(TRADER), 0, "attack: failure is terminal");
        int256 attackTrader = int256(_traderWealth()) - int256(trader0);
        int256 attackLp = _lpNetAssets() - lp0;

        emit log_named_uint("claim-dependent margin (USDG 6dp)", margin);
        emit log_named_int("control: trader change (USDG 6dp)", controlTrader);
        emit log_named_int("control: LP net assets change (USDG 6dp)", controlLp);
        emit log_named_int("attack:  trader change (USDG 6dp)", attackTrader);
        emit log_named_int("attack:  LP net assets change (USDG 6dp)", attackLp);
        emit log_named_int("counterfactual: trader option value (USDG 6dp)", attackTrader - controlTrader);
        emit log_named_int("counterfactual: LP gain forgone (USDG 6dp)", controlLp - attackLp);

        assertLt(controlTrader, -int256(2_900e6), "control: trader bears the 1% adverse move");
        assertGt(attackTrader, -int256(1e6), "attack: trader pays at most the opening bounty");
        assertGt(controlLp, attackLp + int256(2_900e6), "LPs forgo the gain they keep in the control");
    }

    /// @notice The claim is load-bearing: without it, the identical commit is rejected with
    ///         OrderRouter__PredictableOpenInvalid(6) = INSUFFICIENT_INITIAL_MARGIN.
    function test_claimIsTheAdmissionCollateral() public {
        _deferredClaimThenOwnerRecovery();
        uint256 margin = _claimDependentMargin();
        vm.prank(TRADER);
        engine.settleTraderClaim(TRADER);
        vm.expectRevert(abi.encodeWithSignature("OrderRouter__PredictableOpenInvalid(uint8)", uint8(6)));
        this.commitTraderOpen(margin);
    }

    /// @notice When the claimant executes its own order (keepers are permissionless), the opening bounty is released
    ///         back to it: the cancellation costs zero internal USDG (gas and Pyth fees are paid in ETH).
    function test_selfExecutedCancellationCostsNoUsdg() public {
        _deferredClaimThenOwnerRecovery();
        uint64 orderId = _commitOpen(TRADER, CfdTypes.Side.LONG, LOTS, _claimDependentMargin());
        bytes[] memory p1Payload = _publishHistoricalTick(1e8);
        uint256 trader0 = _traderWealth();
        vm.prank(TRADER);
        engine.settleTraderClaim(TRADER);
        vm.deal(TRADER, 1 ether);
        vm.prank(TRADER);
        router.executeOrder(orderId, p1Payload);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "self-executed binding open cancelled");
        assertEq(_traderWealth(), trader0, "zero internal USDG cost");
    }

    // ------------------------------------------------------------------ precondition

    /// @dev Real user flow: matched OI; SHORT wins at the cap and drains pool cash; LONG then wins and its payout
    ///      exceeds free cash, so it is deferred as a senior trader claim and the engine degrades. The owner then
    ///      performs the documented recovery. Result: solvent, not degraded, opens enabled, claim still deferred.
    function _deferredClaimThenOwnerRecovery() internal {
        for (uint256 i = 0; i < 3; i++) {
            _execute(_commitOpen(TRADER, CfdTypes.Side.LONG, 3500, 400_000e6), 1e8);
            _execute(_commitOpen(COUNTERPARTY, CfdTypes.Side.SHORT, 3500, 8_000e6), 1e8);
        }
        _liveMark(2e8);
        _closeAll(COUNTERPARTY, 2e8);
        _liveMark(0.04e8);
        _closeAll(TRADER, 0.04e8);
        assertTrue(engine.degradedMode(), "pre: engine degraded");
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "pre: trader is flat");
        assertGt(engine.traderClaimBalanceAssets(TRADER), 0, "pre: trader holds a deferred claim");

        address owner_ = engine.owner();
        settlementToken.mint(owner_, 3_000_000e6);
        vm.startPrank(owner_);
        settlementToken.approve(address(engine), 3_000_000e6);
        engine.recapitalizePool(3_000_000e6);
        engine.clearDegradedMode();
        vm.stopPrank();
        assertFalse(engine.degradedMode(), "post-recovery: opens enabled");
        assertGt(engine.traderClaimBalanceAssets(TRADER), 0, "post-recovery: claim still deferred");
        _liveMark(1e8);
    }

    /// @dev Smallest marginDelta admitted WITH the claim and rejected WITHOUT it (checked on snapshots).
    function _claimDependentMargin() internal returns (uint256 margin) {
        for (margin = 50e6; margin <= 5_000e6; margin += 25e6) {
            uint256 snap = vm.snapshotState();
            bool withClaim;
            try this.commitTraderOpen(margin) returns (uint64) { withClaim = true; } catch {}
            vm.revertToState(snap);
            if (!withClaim) continue;
            snap = vm.snapshotState();
            vm.prank(TRADER);
            engine.settleTraderClaim(TRADER);
            bool withoutClaim;
            try this.commitTraderOpen(margin) returns (uint64) { withoutClaim = true; } catch {}
            vm.revertToState(snap);
            if (!withoutClaim) return margin;
        }
        revert("no claim-dependent margin");
    }

    function commitTraderOpen(uint256 margin) external returns (uint64) {
        return _commitOpen(TRADER, CfdTypes.Side.LONG, LOTS, margin);
    }

    // ------------------------------------------------------------------ helpers

    function _traderWealth() internal view returns (uint256) {
        return clearinghouse.balanceAssets(TRADER) + engine.traderClaimBalanceAssets(TRADER);
    }

    function _lpNetAssets() internal view returns (int256) {
        return int256(pool.totalAssets()) - int256(engine.totalTraderClaimBalanceAssets());
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory) {
        uint64 w = uint64(router.maxExecutionWindowSeconds());
        return OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + w),
            executionWindowSeconds: uint32(w),
            allowedExecutionModes: 1 | 2 | 4,
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
    }

    function _commitOpen(address a, CfdTypes.Side side, uint256 lots, uint256 margin) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = keccak256(abi.encode("open", a, side, lots, margin, vm.getBlockTimestamp()));
        r.side = side;
        r.sizeDelta = lots * 1e20;
        r.marginDelta = margin;
        r.targetPrice = side == CfdTypes.Side.LONG ? 1 : CAP_PRICE; // permissive open slippage
        r.bounds = _bounds();
        vm.prank(a);
        id = router.commitOrder(r);
    }

    function _execute(uint64 id, uint256 price) internal {
        bytes[] memory u = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrder(id, u);
    }

    function _publishHistoricalTick(uint256 price) internal returns (bytes[] memory u) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 t = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, int32(-8), t, t - 1);
        u = new bytes[](1);
        u[0] = abi.encode(price);
    }

    function _closeAll(address a, uint256 price) internal {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(a);
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = keccak256(abi.encode("close", a, size, vm.getBlockTimestamp()));
        r.side = side;
        r.sizeDelta = size;
        r.targetPrice = side == CfdTypes.Side.LONG ? CAP_PRICE : 1; // permissive close slippage
        r.isClose = true;
        r.bounds = _bounds();
        vm.prank(a);
        uint64 id = router.commitOrder(r);
        bytes[] memory u = _publishHistoricalTick(price);
        vm.prank(KEEPER);
        router.executeOrder(id, u);
    }

    /// @dev Permissionless live mark refresh through the Router.
    function _liveMark(uint256 price) internal {
        uint256 t = vm.getBlockTimestamp();
        uint64 lastMarkTime = engine.lastMarkTime();
        if (t <= lastMarkTime) t = uint256(lastMarkTime) + 1;
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime())
            );
        }
        bytes[] memory d = new bytes[](1);
        d[0] = abi.encode(ticks);
        router.updateMarkPrice{value: 0}(d);
    }
}
```

