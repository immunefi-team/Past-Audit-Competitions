# \#99049 [SC-Critical] LP share price books a refundable VPI charge as junior equity, so an LP who is also the trader redeems at an inflated NAV and takes deposited principal from the LPs who stay

**Submitted on Nov 11th 2026 at 16:46:56 UTC by @kwaro for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99049
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays a volatility/price-impact (VPI) charge in cash to `HousePool`, which books it as `Revenue` so the next reconcile credits it to the junior tranche and it enters the NAV that prices LP redemptions. The **same position** may take that charge straight back as a close-time VPI rebate, paid out of pool cash, and nothing in the reconcile snapshot represents that outstanding obligation. An LP who is also the trader — and, because keepers are permissionless, also the keeper — therefore redeems at a share price inflated by money the pool is about to refund, then collects the refund. On mainnet this takes deposited principal from every liquidity provider who stays: measured end to end on the real contract graph at the release parameters, with the basket flat on every tick, a non-participating junior LP deposited `900000000000` USDG atoms and could withdraw only `899440017375` from its own wallet, against `26` atoms of rounding in a control that runs the identical LP sequence without the trade.

## Vulnerability Details

### Deployed configuration

The program's § Active Features and Networks says severity is assessed with the release parameters of `packages/perps/DEPLOYMENT.md` at this commit. The PoC fixture sets exactly those: `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, maintenance margin 10 bps, initial margin 20 bps, execution fee 4 bps, settlement buffer 25 bps, `baseCarryBps 500`, `minBountyAssets` 1 USDG, `bountyBps 10`, keeper share 50%, senior cap 40,000,000 USDG and 80% of the pool, `minOpenNotionalAssets` 1,000 USDG, `CAP_PRICE 2e8`, 1 USDG seed per tranche, six-feed basket. No arm changes a parameter. The program's § Trusted Roles states plainly that keepers are **not** trusted: "anyone can execute orders, liquidations, price updates and protections".

### Root cause — three facts, each at a line of the pinned source

**1. The charge becomes LP cash immediately.** On an open, the pool-bound part of the trade cost is recorded as revenue — `packages/perps/src/CfdEngineSettlementSidecar.sol:453-464`:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
```

`CashArrived` increments `accountedAssets` (`HousePool.sol:624-640`) and the amount lands in `pendingTradingRevenueAssets`, which the next reconcile distributes through the waterfall to the tranches.

**2. The NAV that prices LP exits has no term for VPI the pool owes back** — `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-79`:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
...
snapshot.distributable = terminalAssets - terminalLiabilities;
```

`physicalAssets` is pool `totalAssets()`, which now includes the charge. The only liabilities subtracted are outstanding trader claims and the terminal **price** delta. `packages/perps/ACCOUNTING_SPEC.md:140-142` states that exclusion as a design choice — "Carry, VPI, execution/frozen spreads, protocol fees, liquidation reserve, order margin, and other action reserves are deliberately excluded: they are separate realized-settlement quantities, not marked price PnL." The charge is excluded as a **liability** while its cash is included as an **asset**.

**3. The same position takes the charge back on close, out of pool cash.** The lifetime clamp permits a negative (rebate) close VPI down to the position's proportional accrual — `packages/perps/src/libraries/CloseAccountingLib.sol:107-118`:

```solidity
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The planner pays it out of free pool cash (`CfdEnginePlanLib.sol:973-977`) and the settlement sidecar executes a real transfer (`CfdEngineSettlementSidecar.sol:621-623`):

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

So the attacker is paid **twice** for one charge: once as an LP, through its share of the revenue the charge created, and once as the trader, through the rebate — and it arranges to be an LP only for the first payment.

### The asymmetry is in the source, not only in the prose

`CfdTypes.sol:41` defines the quantity as signed and two-directional — `int256 vpiAccrued; // [6 dec] Cumulative VPI charges (+) and rebates (-) across the position's lifetime` — yet only the negative branch is ever reserved against. `CfdEngineAccountLens.sol:324-327`:

```solidity
function _negativeVpiReserveTarget(int256 vpiAccruedAssets) ... {
    if (vpiAccruedAssets < 0) {
        targetAssets = uint256(-(vpiAccruedAssets + 1)) + 1;
    }
}
```

`CfdEnginePlanLib.sol:51-53` is the same `if (vpiAccruedAssets < 0)` pattern. There is **no positive-branch counterpart anywhere in `packages/perps/src/`**, and no aggregate of positive `vpiAccrued` across live positions exists in the tree. `packages/perps/SECURITY.md:866-867` states the trader-side rule in words — "gross negative lifetime VPI is backed by a dedicated nonwithdrawable reserve equal to `max(-vpiAccrued, 0)`" — and the direction that matters for an LP exit, positive accrual the pool may have to refund, is backed by nothing. This is the finding, and it rests on code rather than on a reading of prose.

### The sequence — every step an unprivileged real entry point

1. Deposit into the junior tranche (`TrancheVault.requestDeposit` -> epoch settles -> `claimDeposit`).
2. Wait out the 1-hour share cooldown and queue a **full** redemption for the next epoch (`requestRedeem`).
3. 20 seconds before the epoch boundary, open at the 40% pool-relative skew cap through the router (`commitOrder` -> `executeOrder`). The charge is booked: `vpiChargeBooked: 1599998400`, and junior NAV moves `2000000999973` -> `2001600998373` on a flat basket.
4. Settle the LP epoch (`settleLpEpoch`, permissionless) and claim the redemption at that NAV: `attackerLpOut: 1100879998670` on `1100000000000` deposited.
5. Close the position, collect the rebate, withdraw: `attackerTraderOut: 59359784541` of `60000000000` funded, `treasuryCredit: 320000000`, `keeperCredit: 200000`.
6. Net for the round trip: **`attackerNet: 239983211`**.

The victim then performs its own honest exit and receives `victimUsdgOut: 899440017375` — `victimLossVsPar: 559982625` against the `900000000000` it deposited, with `victimSharesLeft: 0`.

### Why the specification does not sanction this

- `packages/perps/ACCOUNTING_SPEC.md:140-142` excludes VPI from the terminal **price** delta, calling it a "separate realized-settlement quantit[y]". That is a statement about marked price PnL. It does not say an exiting LP may be paid a share of a charge the pool is going to refund.
- **The line in the file actually named ACCOUNTING_SPEC does not cover it either**, and I quote it in full because it is the first thing a reader will reach for. `packages/perps/ACCOUNTING_SPEC.md:1057-1066` ends a list of settlement rules with: "anticipated carry, VPI, fees, spreads, and other action economics stay outside terminal `K`; the book is synchronized only to actually installed PnL pledge and same-account claim state after settlement. The VPI reserve therefore does not inflate marked LP receivables or symmetric LP NAV". At a glance that sounds like "VPI is kept out of LP NAV by design". It says two narrower things. *Anticipated* VPI — not yet charged — stays outside terminal `K`, which is the price book, not the tranche NAV. And "the VPI **reserve**" is what the same document defines at `packages/perps/ACCOUNTING_SPEC.md:798` as `vpiRebateReserveAssets = max(-vpiAccrued, 0)` — the **negative** branch, the clawback the pool may be owed. Neither clause says a *realized positive* charge, which the pool may have to refund, stays out of LP NAV. The measurement is that it does not.
- `packages/perps/SECURITY.md:872` says "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV", inside the bullet governing close-time action rebates. I cite it but **do not rest the finding on it, and I flag the ambiguity rather than leave you to find it**: the word "anticipated" appears *exactly once* in `packages/perps/SECURITY.md` (that line) and *zero* times anywhere in `packages/perps/src/`, so it is an undefined phrase with no code referent. It can be read as the pool's anticipated refund or as the trader-side reserve construct. The finding holds under either reading, because the code asymmetry above does not depend on it.
- `packages/perps/WHITEPAPER.md` §5.1 likewise addresses only the trader-side reserve: "LPs cannot withdraw it as marked equity before the matching position obligation settles". This finding is the LP-side mirror of that rule, unenforced.
- § VPI limitations (`packages/perps/SECURITY.md:1434-1444`) lists seven limitations. The closest, "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips", is about a trader extracting a rebate **without** a matching charge. Here the charge is real and fully paid; the defect is that the LP exit price recognises it.

### Is open-time VPI simply LP revenue by design?

This is the one question that, answered the other way, would reduce this to a design disagreement, so the code's answer is put up front. Three facts in `packages/perps/src/` say the design treats a round trip as path-neutral, not as revenue plus a gift:

1. **The quantity is defined as a lifetime net, not a receipt.** `CfdTypes.sol:41` accumulates charges and rebates signed across a position's lifetime. A revenue item is not accumulated signed across a lifetime; a *balance* is.
2. **The clamp exists to bound the refund by the accrual.** `CloseAccountingLib.sol:116` caps a rebate at the charge the position has paid. That is only meaningful if the charge is money the position may get back — nobody clamps a refund against revenue that was never refundable.
3. **The trader side already carries the mirror reserve.** `max(-vpiAccrued, 0)` is held nonwithdrawable. The protocol already accepts that an outstanding lifetime VPI balance must not be spendable by the party that may owe it. This finding is that same rule, unenforced in the direction where the *pool* is the party that may owe it.

If the answer is nonetheless "open-time VPI is LP revenue by design", the late-depositor arm stands on its own: `packages/perps/SECURITY.md:859-860` states as a guarantee that new LPs "do not inherit old trader obligations at a higher deposit NAV", and ARM 6 shows a 500,000 USDG depositor priced at `499875718485` once the rebate lands. That is a stated protocol guarantee failing, independent of how the charge is classified.

### Objections answered in advance, by measurement

**"This is HIGH-1, the known issue about repeatable net-profitable extraction from LPs."** It is not. HIGH-1 is Arbitrum/Pyth clock alignment: `commitTime` comes from the sequencer clock and can trail an already-published Pyth tick, so the qualifying "first post-commit" tick may be known at submission. Its root cause is **oracle timing**. This finding uses none of it — **every arm runs at a flat basket of `1e8` on every tick**, open and close happen at the same price, and the attacker gains nothing from knowing any future price because there is no price movement to know. Swap in a perfectly aligned clock and a trader with no oracle foreknowledge and this finding is unchanged.

**"This is the known depth-manipulation issue."** The two adjacent AI-scan entries (S30, S31) manipulate pool **depth** around a close to change the *magnitude* of its VPI. This finding changes no depth at the close and needs no such move. The axis here is *when the charge is recognised in the LP exit price*, not how large it is.

**"The victim only lost revenue it had not earned."** The measurement rules that out. The victim is compared with its **own deposit**, not with an interim NAV: `900000000000` in, `899440017375` out. The control arm, identical but for the trade, returns `899999999974`.

**"The attacker needs an implausible share of the tranche."** Swept, not derived (junior-only 2,000,000 tranche, position auto-sized to 39% of depth at each open, booked charge `1520998479` at every point):

| attacker share of junior | attacker net | victim loss vs par |
|---|---|---|
| 10% | `-471911520` | `-159911395` (victim **gains**) |
| 20% | `-319812176` | `-7812135` (victim gains) |
| 30% | `-167712953` | `144286980` |
| 40% | `-15613914` | `296385876` |
| 50% | `+136484836` | `448484424` |
| 60% | `+288583071` | `600582357` |
| 65% | `-81051017252` | `550754098` |
| 70% | `-181051017252` | `472074941` |

Three things this establishes, including the one that bounds my own claim. **Break-even is ~41%**, between the 40% and 50% rows, and each 10% of share moves attacker net by `152099848`, exactly 10% of the booked charge — linear in share, as the model predicts. **Below break-even the attack is a donation**: at 10% and 20% the victim ends *above* par. **There is an upper bound too, at ~61%**: at 65% and 70% `attackerLpOut` is the identical `1219572998479`, because the attacker's own redemption cannot be funded while its position is open, stranding `80427` and `180427` USDG of its own stake. The profitable band is therefore **~41% to ~61%** of the junior tranche in this junior-only configuration. (ARM 3 runs 70% of a junior tranche profitably *with a funded senior tranche present*, because senior depth funds the exit, so the bound is configuration-dependent and is not claimed as universal.)

The program's § Feasibility Limitations is explicit that a high capital requirement is "generally not a valid reason to downgrade a report", and that only attacks requiring **USD 100 million or more** are considered case by case. The junior tranche in the arms is 2,000,000 USDG.

**"The rebate is contingent — adverse flow between the open and the close removes it."** This is the strongest objection and it is **partly correct**. VPI is a function of skew *squared*, so the close earns a rebate only while it reduces `|skew|`. With the attacker long ~7,800 lots at 55%, a third-party SHORT landing inside the window:

| third-party SHORT in the window | attacker net |
|---|---|
| 0 lots | `+212534039` |
| 2,000 lots | `+212730135` |
| 4,000 lots | `-1395117339` |
| 6,000 lots | `-3133360554` |
| 7,800 lots | `-4698404700` |

The attack survives adverse flow up to about a quarter of its own size and is destroyed by roughly half, which is the threshold the squared curve predicts. Two measured facts bound the exposure: **the window is 27 seconds** (`openedAt 1709542780`, `closedAt 1709542807`, open to close through the real epoch boundary, settle and claim), and **roughly 400,000 USDG of opposing notional must arrive inside those 27 seconds**, from a party willing to pay its own VPI to do it. I will not overstate the mitigation: the attacker can read the skew before committing the close and simply not close yet, paying carry instead, which converts the loss branch into a delay — that is reasoning, not a measurement, and it is flagged as such rather than claimed as an arm.

**"A guardian can pause and contain it."** § Operating Assumptions says monitoring may reduce severity only where "demonstrated monitoring coverage and response timing establish that containment limits the reported exploit", and that "the existence of monitoring or pause functionality alone does not justify a downgrade". A cycle completes inside one LP epoch, and the LP-settlement hold does not reverse a redemption already claimed.

**"Some existing reserve already accounts for the obligation."** It does not, and this is checkable in one grep. Every guard on the quantity in `packages/perps/src/` tests the **negative** branch only. No aggregate of positive `vpiAccrued` across live positions exists anywhere. `HousePool` holds exactly three buckets apart from distributable value — `pendingClaimantBucketAssets`, `pendingRecapitalizationAssets` and `pendingTradingRevenueAssets` — and the charge is routed into the last, which is the one that *is* distributed.

**"The treasury takes most of the extracted value, so the attacker is not stealing."** The impact row is about the victim's funds, not the attacker's margin. The attacker is nonetheless net positive in every exploit arm (`239983211`, `934399054`, `5144293532` cumulative over 24 cycles), so this is not griefing under the program's own spend-to-damage test either.

## Impact Details

**The quantity is settlement-token (USDG) principal that a non-participating liquidity provider can withdraw from the junior tranche**, measured in its own wallet after a complete `requestRedeem` -> epoch settlement -> `claimRedeem` exit — not an interim NAV reading, and not unclaimed yield. The basket is `1e8` on every tick of every arm, so no part of the loss comes from a market move.

| arm | attacker net | victim, measured | control |
|---|---|---|---|
| junior only, attacker 1,100,000 / victim 900,000 USDG, 8,000 lots | **+239983211** | withdrew **899440017375** of **900000000000** deposited | victim withdrew `899999999974`; attacker `-10` |
| senior 2,000,000 present, attacker 1,400,000 / victim 600,000 USDG, 16,000 lots | **+934399054** | NAV `599994429221` -> `598389013040` | attacker `-25570783` |
| three cycles, same victim, 32,412 simulated seconds | **+637842443** cumulative | withdrew **898426479206** | — |
| a late depositor's 500,000,000,000 entry priced in the pulsed epoch | — | shares worth **499875718485** | — |

The attacker's gain is *smaller* than the victim's loss because the protocol treasury takes `320000000` of it in execution fees. The impact row thresholds the victim's funds, not the attacker's profit, which is why the loss is the measure of the theft.

**It is not a one-off, and the sustained rate is measured rather than extrapolated.** 24 cycles against one victim who deposits once and never acts again:

| quantity | measured |
|---|---|
| elapsed | `259212` seconds — exactly 72 hours |
| victim deposited | `900000000000` |
| victim withdrew, after a complete exit | `887390839812` |
| victim loss vs par | **`12609160188`** |
| as a fraction of its own principal | **`14010` ppm = 1.4010% in three days** |
| the rate the run itself computes | **`4669` ppm/day = 0.4669% of LP principal per day** |
| cumulative attacker net | `5144293532` |
| junior depth, first open -> last | `2000002000000` -> `1987919091415` |
| position size, first open -> last | `7800` lots -> `7752` lots |

Three things follow from those numbers rather than from argument. **The attack does not self-extinguish**: after 72 hours the pool is 0.6% shallower and the position the attacker can open has shrunk by 48 lots out of 7,800, so the next cycle is worth 99.4% of the last. **The attacker's capital requirement is not cumulative** — the same `1100000000000` is redeemed and redeposited every cycle, so 24 cycles cost no more capital than one. And **the extraction is linear in TVL**: at ten times the tranche (junior 20,000,000) the charge is `15209998479`, attacker net `2125351761` and victim loss `5245351215` on a `9000000000000` deposit — the same fraction, with the attacker's own outlay `11000000000000`, that is 11 million USDG, two orders of magnitude below the USD 100 million at which the program's § Attack Investment Amount says a capital requirement is even considered case by case.

**There is a second victim class.** An LP that is not present beforehand is damaged too: a 500,000 USDG deposit priced into the pulsed epoch is worth `499875718485` once the rebate lands. `packages/perps/SECURITY.md:859-860` states as a guarantee that this cannot happen — new LPs "do not inherit old trader obligations at a higher deposit NAV".

**Severity.** The selected impact is **"Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield"**, which this program's published table rates **Critical**. The loss is principal — the victim ends below its own deposit, measured in its wallet, with no market move and no action of its own — so it is not the adjacent "Theft of unclaimed yield" row. The feasibility bounds disclosed above (the ~41% to ~61% capital band, the 27-second window, the adverse-flow contingency) are **feasibility limitations**, and the program's own rule governs what they may affect: *"Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it. A finding with a large impact that is infeasible retains that impact and severity, but is rewarded at a reduced amount."* The adjacent rules add that attacker risk justifies a downgrade only "where the risk massively outweighs the reward", and that "low return on investment alone is not sufficient".

## References

Pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Paths are relative to the repository root.

Primary defect:
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-79` — the reconcile snapshot, with no liability term for refundable VPI
- `packages/perps/src/HousePool.sol:18` imports it and calls `buildReconcileSnapshot` at `:1144`, `:1453-1454`, `:1562-1563`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:453-464` — the charge recognised as `Revenue` / `CashArrived`
- `packages/perps/src/HousePool.sol:624-640` — `CashArrived` incrementing `accountedAssets`
- `packages/perps/src/libraries/CloseAccountingLib.sol:107-118` — the lifetime clamp permitting the rebate
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:973-977` and `packages/perps/src/CfdEngineSettlementSidecar.sol:621-623` — the rebate paid from pool cash
- `packages/perps/src/CfdTypes.sol:41` — `vpiAccrued` declared signed and two-directional
- `packages/perps/src/CfdEngineAccountLens.sol:324-327` and `packages/perps/src/libraries/CfdEnginePlanLib.sol:51-53` — the negative-only reserve, with no positive counterpart

Specification:
- `packages/perps/ACCOUNTING_SPEC.md:140-142`, `:798`, `:1057-1066`
- `packages/perps/SECURITY.md:859-860`, `:866-867`, `:872`, `:1434-1444`
- `packages/perps/WHITEPAPER.md` §5.1
- `packages/perps/DEPLOYMENT.md` — the release parameters the arms use

Prior art in this repository, each already an ancestor of the pin, showing that NAV failing to carry an obligation is a defect class this protocol recognises and fixes rather than intended economics:
- commit `13ba8903` — "Fix C-01 & C-02: stale mark price NAV arbitrage and ghost funding liability"
- commit `d752afe6` (PR #39) — branch `codex/fix-deposit-nav-loss-blindness`
- commit `16240409` (PR #62) — branch `codex/junior-unfair`

## Suggested fix

Do not remove the rebate — the differential probe below does that, and it confiscates the legitimate symmetric rebate a skew-reducing close has earned. The asymmetry to close is the recognition, on the LP side:

1. **Carry a liability for outstanding refundable VPI in the reconcile snapshot.** `HousePool`'s engine snapshot already receives `traderClaimBalanceAssets` and the terminal price delta; add the aggregate of positive lifetime `vpiAccrued` across live positions (the amount the clamp would permit to be rebated) and subtract it in `HousePoolAccountingLib.buildReconcileSnapshot`, exactly as the trader side already reserves `max(-vpiAccrued, 0)` in a dedicated nonwithdrawable bucket.
2. **Or defer recognition.** Route the open's VPI component to a pending bucket that the waterfall only distributes once the matching position settles, in the way `pendingRecapitalizationAssets` is already held apart from distributable revenue. The execution fee, which is not refundable, can continue to be recognised at once.

Either closes the exit-price gap without touching the VPI curve, the clamp, or the trader-side reserve.



## Link to Proof of Concept

https://gist.github.com/ibrahimatix0x01/c3fba67cb907ea5d188e0cd141c82819

## Proof of Concept

### How to run

```
# from the repository root, at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8
cd packages/perps
mkdir -p test/perps/hunt4
# copy the four .sol files embedded below into test/perps/hunt4/
git diff --quiet -- src/          # MUST be clean: no source file is modified

forge test --force --match-path 'test/perps/hunt4/F42_VpiNavPulse.t.sol'   -vv   # the 6 exploit/control arms
forge test --force --match-path 'test/perps/hunt4/F42_Feasibility.t.sol'   -vv   # the feasibility sweeps
```

Last run at the pin with `src/` byte-identical: **23 tests passed, 0 failed, 0 skipped**, exit 0.

Only two things are mocked, both expressly permitted by the program and both the vendor's own pristine test utilities: Pyth (`MockPyth`, since signed Wormhole-attested payloads cannot be produced locally) and USDG (`MockSettlementToken`, since Global Dollar is not deployed in a local Foundry run). Every contract on the path under test is real: `HousePool`, `TrancheVault`, `CfdEngine`, the planner, the settlement sidecar, the clearinghouse, the router, the lifecycle book, the terminal NAV book and the oracle wrapper. There is **no `vm.store`**, no `deal`/`mint`/`burn` against a protocol address (USDG is minted to user EOAs, ETH is dealt to actors only for the Pyth fee), no `prank` as a protocol contract, and no owner action beyond deployment. The attacker is also the keeper, which § Trusted Roles expressly allows.

### Step by step — ARM 1, `test_F42_exploit_observedVictimWallet`

The junior tranche holds 2,000,000 USDG: the victim 900,000 and the attacker 1,100,000. The basket is `1e8` on every tick throughout.

1. **Both LPs enter honestly.** Victim and attacker each call `TrancheVault.requestDeposit`, the epoch is settled with `OrderRouter.settleLpEpoch`, and both call `claimDeposit`. The victim now holds shares it will not touch again until step 8.
2. **The attacker queues its exit.** After the 1-hour share cooldown it calls `requestRedeem` for its **entire** balance, targeting epoch `E`. Nothing about this is unusual or restricted.
3. **20 seconds before the boundary of `E`, the attacker opens at the skew cap.** `commitOrder` then `executeOrder` for LONG 8,000 lots with 40,000 USDG margin, sized just inside the 40% pool-relative `maxSkewRatio`. Because the open *increases* skew, it pays a VPI charge: the test asserts `vpiChargeBooked: 1599998400` by reading `engine.positions(...)`, and requires it to be strictly positive.
4. **The charge is now junior equity.** Junior NAV moves `2000000999973` -> `2001600998373` — the entire `1599998400`, on a flat basket, with no price movement anywhere.
5. **The attacker settles the epoch itself.** `settleLpEpoch` is permissionless, so no third party or timing luck is needed. The redemption is priced at the inflated NAV.
6. **The attacker claims the redemption**: `attackerLpOut: 1100879998670` against the `1100000000000` it deposited. It has just been paid its share of a charge the pool is about to refund.
7. **The attacker closes and collects the refund.** The close *reduces* skew, so the lifetime clamp permits a negative (rebate) VPI, paid from pool cash. After withdrawing: `attackerTraderOut: 59359784541` of `60000000000` funded, `treasuryCredit: 320000000`, `keeperCredit: 200000`. Round-trip net: **`attackerNet: 239983211`**.
8. **The victim exits honestly and is short.** `requestRedeem` -> epoch settles -> `claimRedeem`, leaving `victimSharesLeft: 0`. Its wallet holds **`899440017375`** against the `900000000000` it deposited: **`victimLossVsPar: 559982625`**, taken from principal, with the basket never having moved.

### The control, `test_F42_control_observedVictimWallet`

The identical LP sequence with **no trade at all**: same deposits, same cooldown, same epoch, same redemption. The victim withdraws `899999999974` — a loss of **26 atoms** of rounding — and the attacker nets `-10`. The only difference between the arms is the open/close round trip, so the `559982625` shortfall is attributable to it and to nothing else.

### Differential probe — proof the arms are sensitive to this one branch

The one-statement edit below is applied to the pinned source and the same arms re-run. It is **not** the recommended remediation (it also confiscates the legitimate symmetric rebate a skew-reducing close has earned); its only job is to isolate the branch under test:

```solidity
// src/libraries/CloseAccountingLib.sol, immediately after the lifetime clamp
if (state.vpiDeltaAssets < 0) { state.vpiDeltaAssets = 0; }
```

| | at the pin | with the probe |
|---|---|---|
| `attackerNet` (ARM 1) | `239983211` | `-1360015189` |
| `victimUsdgOut` (ARM 1) | `899440017375` | `900720014352` |
| `victimLossVsPar` (ARM 1) | `559982625` | `-720014352` |
| ARM 3 attacker net | `934399054` | `-2265599346` |
| ARM 5 cumulative net | `637842443` | `-3927337801` |
| ARM 6 late-depositor share value | `499875718485` | `500000001561` |
| both control arms | pass | pass, gas identical (`6155529`, `6535382`) |

Under the probe the victim **gains** `720014352`: the charge it was credited is no longer refunded away. Both controls are untouched at identical gas, so the probe closes the bug rather than breaking the build.

### The arms

| arm | what it establishes |
|---|---|
| `test_F42_exploit_observedVictimWallet` | the headline, measured as wallet USDG after a real exit |
| `test_F42_control_observedVictimWallet` | the identical LP sequence without the trade: victim loss 26 atoms, attacker net `-10` |
| `test_F42_exploit_seniorPresent` | the same mechanism with a senior tranche, the largest absolute numbers |
| `test_F42_control_seniorPresent` | its control: attacker net `-25570783` |
| `test_F42_exploit_threeCyclesRepeatable` | repeatable: cumulative net `637842443`, victim loss `1573520794` |
| `test_F42_exploit_lateDepositorBuysInflatedNav` | a second victim class: a fresh depositor priced into the pulse |
| `test_F42F_control_driverMatchesProvenArm` | asserts ARM 1's three published numbers through the feasibility file's own copy of the driver, so a drifted copy voids every sweep rather than reporting silently |
| `test_F42F_sweepA_*` | the break-even curve, 10% to 70% of the junior tranche |
| `test_F42F_sweepB_*` | adverse third-party flow landing inside the window |
| `test_F42F_armW_windowSeconds` | the window, measured end to end: 27 seconds |
| `test_F42F_armS_deployedScale` | ten times the tranche; every quantity linear |
| `test_F42F_armR_sustainedRate` | 24 cycles over 72 measured hours; the per-day drain rate |

---

## Full source

Four files, dropped into `packages/perps/test/perps/hunt4/`. Nothing else is added and **no file under `src/` is modified**.

### `test/perps/hunt4/F42Fixture.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// plether-cycle2 F0  --  the RELEASE fixture.
// Deploys the real contract graph exactly as script/DeployPerpsArbitrumSepolia.s.sol:197-427 does, with the
// release wrappers (script lines 71-131) reproduced verbatim, the six-feed basket (script lines 147-166, 464-500),
// release RiskParams (script lines 434-447), senior caps 40M / 8000, minOpenNotional 1000e6, 1 USDG seeds per
// tranche. Only Pyth (MockPyth, packages/shared/test-support) and USDG (MockSettlementToken) are mocked  --  the two
// out-of-scope components the policy's PoC section allows to be mocked. Nothing under packages/perps/src is touched.

import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAccountLens} from "@plether/perps/CfdEngineAccountLens.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineProtocolLens} from "@plether/perps/CfdEngineProtocolLens.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";

/// @dev Verbatim copy of script/DeployPerpsArbitrumSepolia.s.sol:73-83.
contract ReleaseHousePool is HousePool {

    constructor(
        address settlementToken,
        address engine,
        address redemptionMathSidecar
    ) HousePool(settlementToken, engine, redemptionMathSidecar) {
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        _validatePoolConfig(poolConfig);
    }

}

/// @dev Verbatim copy of script/DeployPerpsArbitrumSepolia.s.sol:89-101.
contract ReleaseOracle is PletherOracle {

    constructor(
        address engine,
        address housePool,
        address pyth,
        bytes32[] memory feedIds,
        uint256[] memory quantities,
        uint256[] memory basePrices,
        bool[] memory inversions
    ) PletherOracle(engine, housePool, pyth, feedIds, quantities, basePrices, inversions) {
        adverseConfidenceMultiplierBps = 2500;
    }

}

/// @dev Verbatim copy of script/DeployPerpsArbitrumSepolia.s.sol:107-131.
contract ReleaseRouter is OrderRouter {

    constructor(
        address engine,
        address engineLens,
        address housePool,
        address pletherOracle_,
        address keeperSidecar,
        address policyEvaluator,
        address executionSidecar,
        address lifecycleBook
    )
        OrderRouter(
            engine, engineLens, housePool, pletherOracle_, keeperSidecar, policyEvaluator, executionSidecar, lifecycleBook
        )
    {
        minOpenNotionalAssets = 1000e6;
    }

}

/// @dev Every clock read in this harness goes through vm.getBlockTimestamp()/vm.getBlockNumber(): under via_ir the
///      optimizer treats block fields as constant within one transaction even across cheatcodes (the vendor's own
///      note at BasePerpTest.sol:304), and a cached timestamp produced a historical payload for the WRONG commit.
abstract contract ReleaseFixture is Test {

    // ---- release constants (script/DeployPerpsArbitrumSepolia.s.sol:136-166) ----
    uint32 internal constant CAP_PRICE = 2e8;
    uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 50;
    uint256 internal constant JUNIOR_MAINTENANCE_FEE_APR_BPS = 0;
    uint256 internal constant SEED_ASSETS = 1e6;
    uint256 internal constant FEEDS = 6;

    bytes32 internal constant PYTH_EUR_USD = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
    bytes32 internal constant PYTH_USD_JPY = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
    bytes32 internal constant PYTH_GBP_USD = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
    bytes32 internal constant PYTH_USD_CAD = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
    bytes32 internal constant PYTH_USD_SEK = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
    bytes32 internal constant PYTH_USD_CHF = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;

    /// @dev Monday 2024-03-04 10:00 UTC  --  a live weekday, standard time (DST begins 2024-03-10).
    uint256 internal constant SETUP_TIMESTAMP = 1_709_532_000;

    bytes32[] internal feedIds;
    uint256[] internal quantities;
    uint256[] internal basePrices;
    bool[] internal inversions;

    // ---- the graph ----
    MockSettlementToken internal usdg;
    MockPyth internal pyth;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    TerminalNavBookV2 internal terminalNavBook;
    CfdEnginePlanner internal planner;
    CfdEngineSettlementSidecar internal settlementSidecar;
    CfdEngineAdmin internal engineAdmin;
    HousePoolRedemptionMathSidecar internal redemptionMathSidecar;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    CfdEngineAccountLens internal accountLens;
    CfdEngineLens internal engineLens;
    CfdEngineProtocolLens internal protocolLens;
    PletherOracle internal oracle;
    CfdOrderPolicyEvaluator internal policyEvaluator;
    OrderRouterExecutionSidecar internal executionSidecar;
    OrderLifecycleBook internal lifecycleBook;
    OrderRouterLiquidationBatchSidecar internal keeperSidecar;
    OrderRouter internal router;
    OrderRouterAdmin internal routerAdmin;
    IPositionProtectionBook internal protectionBook;
    EmergencyPauseCoordinator internal pauseCoordinator;

    /// @dev Pyth fee per update call. Nonzero on purpose so every ETH refund / deferral lane is live.
    uint256 internal constant PYTH_FEE = 1 gwei;

    receive() external payable {}

    // =====================================================================================================
    // setUp  --  script/DeployPerpsArbitrumSepolia.s.sol:197-427, in the same order
    // =====================================================================================================

    function setUp() public virtual {
        vm.warp(SETUP_TIMESTAMP);
        vm.roll(1_000_000);
        _initBasketConfig();

        usdg = new MockSettlementToken();
        pyth = new MockPyth();
        pyth.setFee(PYTH_FEE);

        clearinghouse = new MarginClearinghouse(address(usdg));
        engine = new CfdEngine(address(usdg), address(clearinghouse), CAP_PRICE, _riskParams(), FROZEN_CLOSE_SPREAD_BPS);
        terminalNavBook = new TerminalNavBookV2(address(engine), CAP_PRICE);
        engine.setTerminalNavBook(address(terminalNavBook));

        planner = new CfdEnginePlanner();
        settlementSidecar = new CfdEngineSettlementSidecar(address(engine));
        engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlementSidecar), address(engineAdmin));

        redemptionMathSidecar = new HousePoolRedemptionMathSidecar();
        pool = new ReleaseHousePool(address(usdg), address(engine), address(redemptionMathSidecar));
        seniorVault =
            new TrancheVault(IERC20(address(usdg)), address(pool), true, "Plether Senior LP", "psLP", 0, address(0));
        pool.setSeniorVault(address(seniorVault));
        address juniorFeeRecipient = engine.protocolTreasury();
        require(juniorFeeRecipient != address(0), "treasury zero");
        juniorVault = new TrancheVault(
            IERC20(address(usdg)),
            address(pool),
            false,
            "Plether Junior LP",
            "pjLP",
            JUNIOR_MAINTENANCE_FEE_APR_BPS,
            juniorFeeRecipient
        );
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        accountLens = new CfdEngineAccountLens(address(engine));
        engineLens = new CfdEngineLens(address(engine));
        protocolLens = CfdEngineProtocolLens(address(pool.ENGINE_PROTOCOL_LENS()));
        oracle = new ReleaseOracle(
            address(engine), address(pool), address(pyth), feedIds, quantities, basePrices, inversions
        );
        policyEvaluator = new CfdOrderPolicyEvaluator();
        executionSidecar = new OrderRouterExecutionSidecar();
        uint64 nonce = vm.getNonce(address(this));
        address expectedRouter = vm.computeCreateAddress(address(this), uint256(nonce) + 2);
        lifecycleBook =
            new OrderLifecycleBook(expectedRouter, address(engine), address(clearinghouse), address(pool));
        keeperSidecar = new OrderRouterLiquidationBatchSidecar(expectedRouter);
        require(oracle.adverseConfidenceMultiplierBps() == 2500, "adverse multiplier");
        router = new ReleaseRouter(
            address(engine),
            address(engineLens),
            address(pool),
            address(oracle),
            address(keeperSidecar),
            address(policyEvaluator),
            address(executionSidecar),
            address(lifecycleBook)
        );
        require(address(router) == expectedRouter, "router CREATE address");
        protectionBook = router.positionProtectionBook();
        routerAdmin = OrderRouterAdmin(router.admin());

        // script:449-464  --  the release config the deploy verifies before wiring
        require(pool.maxSeniorExposureAssets() == 40_000_000e6, "senior exposure cap");
        require(pool.maxSeniorShareBps() == 8000, "senior share cap");
        require(router.minOpenNotionalAssets() == 1000e6, "min open notional");
        require(router.pletherOracle().basketMaxConfidenceRatioBps() == 10, "basket confidence");
        require(router.maxPendingOrders() == 5, "max pending orders");

        engine.setOrderRouter(address(router));
        clearinghouse.setEngine(address(engine));

        pauseCoordinator = new EmergencyPauseCoordinator(address(routerAdmin), address(pool), address(this));
        pool.setPauser(address(pauseCoordinator));
        routerAdmin.setPauser(address(pauseCoordinator));

        // Release README: trading is inactive until both seeds exist and activateTrading() is called.
        _installFirstMark(1e8);
        _seedTranche(false);
        _seedTranche(true);
        pool.activateTrading();
    }

    function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
        // script/DeployPerpsArbitrumSepolia.s.sol:434-447
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

    function _initBasketConfig() internal {
        feedIds = new bytes32[](FEEDS);
        feedIds[0] = PYTH_EUR_USD;
        feedIds[1] = PYTH_USD_JPY;
        feedIds[2] = PYTH_GBP_USD;
        feedIds[3] = PYTH_USD_CAD;
        feedIds[4] = PYTH_USD_SEK;
        feedIds[5] = PYTH_USD_CHF;
        quantities = new uint256[](FEEDS);
        quantities[0] = 576 * 10 ** 15;
        quantities[1] = 136 * 10 ** 15;
        quantities[2] = 119 * 10 ** 15;
        quantities[3] = 91 * 10 ** 15;
        quantities[4] = 42 * 10 ** 15;
        quantities[5] = 36 * 10 ** 15;
        basePrices = new uint256[](FEEDS);
        basePrices[0] = 117_500_000;
        basePrices[1] = 638_000;
        basePrices[2] = 134_480_000;
        basePrices[3] = 72_880_000;
        basePrices[4] = 10_860_000;
        basePrices[5] = 126_100_000;
        inversions = new bool[](FEEDS);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
    }

    function _seedTranche(bool senior) internal {
        usdg.mint(address(this), SEED_ASSETS);
        usdg.approve(address(pool), SEED_ASSETS);
        pool.initializeSeedPosition(senior, SEED_ASSETS, address(this));
    }

    function _installFirstMark(uint256 basket8) internal {
        vm.deal(address(this), 1 ether);
        router.updateMarkPrice{value: PYTH_FEE}(_livePayload(basket8));
    }

    // =====================================================================================================
    // Six-feed payload builders. A basket price B (8 dec) is realised by setting every direct feed to
    // base_i * B / 1e8 and every inverted feed to round(1e24 / (base_i * B)), which PletherOracle normalises
    // back to base_i * B / 1e8 (PletherOracle.sol:1008-1037). Component i then contributes q_i * B / 1e18.
    // =====================================================================================================

    struct FeedTiming {
        uint64[6] publishTime;
        uint64[6] prevPublishTime;
    }

    function _componentRaw(uint256 i, uint256 basket8) internal view returns (int64) {
        if (!inversions[i]) {
            return int64(uint64(basePrices[i] * basket8 / 1e8));
        }
        uint256 denom = basePrices[i] * basket8;
        return int64(uint64((1e24 + denom / 2) / denom));
    }

    /// @dev conf expressed in bps of each component's own price; the oracle caps the AGGREGATE at 10 bps.
    function _payload(
        uint256 basket8,
        uint256 confBps,
        FeedTiming memory timing
    ) internal view returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](FEEDS);
        for (uint256 i; i < FEEDS; ++i) {
            int64 raw = _componentRaw(i, basket8);
            uint64 conf = uint64(uint256(uint64(raw)) * confBps / 10_000);
            ticks[i] = MockPyth.FeedUpdate(
                feedIds[i], MockPyth.MockPrice(raw, conf, -8, timing.publishTime[i], timing.prevPublishTime[i])
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _uniformTiming(uint64 publishTime, uint64 prevPublishTime) internal pure returns (FeedTiming memory t) {
        for (uint256 i; i < FEEDS; ++i) {
            t.publishTime[i] = publishTime;
            t.prevPublishTime[i] = prevPublishTime;
        }
    }

    /// @dev A live tick for every feed at `vm.getBlockTimestamp()`.
    function _livePayload(uint256 basket8) internal view returns (bytes[] memory) {
        uint64 now_ = uint64(vm.getBlockTimestamp());
        return _payload(basket8, 0, _uniformTiming(now_, now_ - 1));
    }

    /// @dev The unique post-commit tick an order executes against: publishTime = commitTime + 1 for every feed.
    function _histPayload(uint256 basket8, uint64 commitTime) internal view returns (bytes[] memory) {
        return _payload(basket8, 0, _uniformTiming(commitTime + 1, commitTime));
    }

    // =====================================================================================================
    // Real-entry-point drivers
    // =====================================================================================================

    function _advanceBlock(uint256 seconds_) internal {
        vm.warp(vm.getBlockTimestamp() + seconds_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _refreshMark(address keeper, uint256 basket8) internal {
        vm.deal(keeper, keeper.balance + PYTH_FEE);
        vm.prank(keeper);
        router.updateMarkPrice{value: PYTH_FEE}(_livePayload(basket8));
    }

    function _fundTrader(address trader, uint256 amount) internal {
        usdg.mint(trader, amount);
        vm.startPrank(trader);
        usdg.approve(address(clearinghouse), amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _openBounds() internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + 60);
        b.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        b.allowedExecutionModes = 7;
        b.expectedConfigHash = lifecycleBook.currentExecutionConfigHash();
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

    function _request(
        bytes32 clientId,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = clientId;
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        // LONG profits when the basket FALLS (CfdTypes.sol:14-19), so an open LONG wants a high entry: its limit is a
        // floor (1 = accept anything) and an open SHORT's limit is a ceiling (CAP). Closes invert that.
        // Same convention as LegacyOrderRouterHarness.sol:52 and OrderValidationLib.checkSlippage.
        r.targetPrice = isClose
            ? (side == CfdTypes.Side.LONG ? uint256(CAP_PRICE) : 1)
            : (side == CfdTypes.Side.LONG ? 1 : uint256(CAP_PRICE));
        r.isClose = isClose;
        r.bounds = _openBounds();
    }

    uint256 internal clientNonce;

    function _nextClientId() internal returns (bytes32) {
        return bytes32(++clientNonce);
    }

    function _commit(address trader, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        returns (uint64 orderId, uint64 commitTime)
    {
        OrderV3Types.OrderRequest memory r = _request(_nextClientId(), side, size, margin, isClose);
        vm.prank(trader);
        orderId = router.commitOrder(r);
        commitTime = uint64(vm.getBlockTimestamp());
    }

    function _execute(address keeper, uint64 orderId, uint64 commitTime, uint256 basket8)
        internal
        returns (OrderV3Types.ExecutionResult memory result)
    {
        bytes[] memory data = _histPayload(basket8, commitTime);
        uint256 fee = oracle.getOrderExecutionFee(data);
        vm.deal(keeper, keeper.balance + fee);
        vm.prank(keeper);
        result = router.executeOrder{value: fee}(orderId, data);
    }

    function _liquidate(address keeper, address account, uint256 basket8) internal {
        bytes[] memory data = _livePayload(basket8);
        vm.deal(keeper, keeper.balance + PYTH_FEE);
        vm.prank(keeper);
        router.executeLiquidation{value: PYTH_FEE}(account, data);
    }

    function _settleLpEpoch(address keeper, uint256 basket8) internal {
        bytes[] memory data = _livePayload(basket8);
        vm.deal(keeper, keeper.balance + PYTH_FEE);
        vm.prank(keeper);
        router.settleLpEpoch{value: PYTH_FEE}(data);
    }

    /// @dev Full async LP deposit through the vault: request → wait for the epoch → router settlement → claim.
    function _fundTranche(TrancheVault vault, address lp, uint256 amount, address keeper, uint256 basket8)
        internal
        returns (uint256 shares)
    {
        usdg.mint(lp, amount);
        vm.startPrank(lp);
        usdg.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activation = vault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activation) {
            vm.warp(activation);
            vm.roll(vm.getBlockNumber() + 1);
        }
        _settleLpEpoch(keeper, basket8);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _logResult(string memory tag, OrderV3Types.ExecutionResult memory r) internal {
        emit log_named_uint(string.concat(tag, ".orderId"), r.orderId);
        emit log_named_uint(string.concat(tag, ".status 1=Pending 2=Executed 3=Failed"), uint8(r.status));
        emit log_named_uint(string.concat(tag, ".terminalReason"), uint8(r.terminalReason));
        emit log_named_uint(string.concat(tag, ".pendingReason"), uint8(r.pendingReason));
    }

    function _positionSize(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _protectionParams(uint256 tp, uint256 sl)
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory p)
    {
        p.takeProfitTriggerPrice = tp;
        p.stopLossTriggerPrice = sl;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _rawUsdgInProtocol() internal view returns (uint256) {
        return usdg.balanceOf(address(pool)) + usdg.balanceOf(address(clearinghouse))
            + usdg.balanceOf(address(seniorVault)) + usdg.balanceOf(address(juniorVault)) + usdg.balanceOf(address(engine));
    }

}
```

### `test/perps/hunt4/F42Base.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// F4-2 PoC shared drivers (from the hunt's lane-B rig, unchanged in behaviour). Everything goes through real entry points
// (TrancheVault requests/claims, OrderRouter.settleLpEpoch / commitOrder / executeOrder / executeLiquidation,
// MarginClearinghouse.deposit / withdraw). No vm.store, no deal/mint/burn against a protocol address.

import {ReleaseFixture} from "./F42Fixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

abstract contract F42Base is ReleaseFixture {

    uint256 internal constant LOT = 1e20; // CfdTypes.SIZE_QUANTUM: 100 tokens, 100 USDG notional at basket 1e8
    address internal constant KEEPER = address(0x4EE9);

    // ---------------------------------------------------------------- time
    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _epoch() internal view returns (uint256) {
        return vm.getBlockTimestamp() / 1 hours;
    }

    function _warpTo(uint256 ts) internal {
        if (vm.getBlockTimestamp() < ts) {
            vm.warp(ts);
            vm.roll(vm.getBlockNumber() + 1);
        }
    }

    function _warpToEpoch(uint256 epochId) internal {
        _warpTo(epochId * 1 hours);
    }

    // ---------------------------------------------------------------- LP drivers
    function _mintApprove(address lp, TrancheVault vault, uint256 amount) internal {
        usdg.mint(lp, amount);
        vm.prank(lp);
        usdg.approve(address(vault), type(uint256).max);
    }

    function _reqDeposit(TrancheVault vault, address lp, uint256 amount) internal returns (uint256 requestId) {
        _mintApprove(lp, vault, amount);
        vm.prank(lp);
        requestId = vault.requestDeposit(amount, lp, lp);
    }

    function _claimAllDeposit(TrancheVault vault, address lp, uint256 requestId) internal returns (uint256 shares) {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _reqRedeem(TrancheVault vault, address lp, uint256 shares) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    function _claimAllRedeem(TrancheVault vault, address lp, uint256 requestId) internal returns (uint256 assets) {
        uint256 claimable = vault.claimableRedeemRequest(requestId, lp);
        if (claimable == 0) {
            return 0;
        }
        vm.prank(lp);
        assets = vault.claimRedeem(requestId, claimable, lp, lp);
    }

    /// @dev Settlement that is allowed to make no progress (returns false instead of reverting the test).
    function _trySettle(address keeper, uint256 basket8) internal returns (bool ok, bytes memory err) {
        bytes[] memory data = _livePayload(basket8);
        vm.deal(keeper, keeper.balance + PYTH_FEE);
        vm.prank(keeper);
        (ok, err) = address(router).call{value: PYTH_FEE}(abi.encodeWithSelector(router.settleLpEpoch.selector, data));
    }

    // ---------------------------------------------------------------- trader drivers
    function _open(address trader, CfdTypes.Side side, uint256 lots, uint256 margin, uint256 basket8)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        (uint64 id, uint64 ct) = _commit(trader, side, lots * LOT, margin, false);
        _advanceBlock(2);
        r = _execute(KEEPER, id, ct, basket8);
    }

    function _close(address trader, CfdTypes.Side side, uint256 lots, uint256 basket8)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        (uint64 id, uint64 ct) = _commit(trader, side, lots * LOT, 0, true);
        _advanceBlock(2);
        r = _execute(KEEPER, id, ct, basket8);
    }

    function _withdrawAll(address trader) internal returns (uint256 amount) {
        amount = clearinghouse.balanceAssets(trader);
        if (amount == 0) {
            return 0;
        }
        vm.prank(trader);
        clearinghouse.withdraw(trader, amount);
    }

    // ---------------------------------------------------------------- views
    function _nav() internal view returns (uint256 seniorNav, uint256 juniorNav) {
        (seniorNav, juniorNav,,) = pool.getPendingTrancheState();
    }

    /// @dev Vault asset escrow identity: token balance == pendingDepositEscrow + withdrawalEscrow (no donation in test).
    function _assetEscrowGap(TrancheVault vault) internal view returns (int256) {
        return int256(usdg.balanceOf(address(vault)))
            - int256(vault.pendingDepositEscrowAssets() + vault.withdrawalEscrowAssets());
    }

    /// @dev Vault share escrow identity: share balance == depositClaimEscrow + pendingRedeemEscrow.
    function _shareEscrowGap(TrancheVault vault) internal view returns (int256) {
        return int256(vault.balanceOf(address(vault)))
            - int256(vault.depositClaimEscrowShares() + vault.pendingRedeemEscrowShares());
    }

    function _logPool(string memory tag) internal {
        (uint256 s, uint256 j) = _nav();
        emit log_named_uint(string.concat(tag, " pool.raw"), usdg.balanceOf(address(pool)));
        emit log_named_uint(string.concat(tag, " pool.accounted"), pool.accountedAssets());
        emit log_named_uint(string.concat(tag, " seniorPrincipal(stored)"), pool.seniorPrincipal());
        emit log_named_uint(string.concat(tag, " juniorPrincipal(stored)"), pool.juniorPrincipal());
        emit log_named_uint(string.concat(tag, " seniorNAV(pending)"), s);
        emit log_named_uint(string.concat(tag, " juniorNAV(pending)"), j);
        emit log_named_uint(string.concat(tag, " seniorHWM"), pool.seniorHighWaterMark());
        emit log_named_uint(string.concat(tag, " unassigned"), pool.unassignedAssets());
        emit log_named_uint(string.concat(tag, " reservedSeniorDeposit"), pool.reservedSeniorDepositAssets());
    }

}
```

Here is the remaining part of the POC in a secret gist as the field can not contain it all:

https://gist.github.com/ibrahimatix0x01/c3fba67cb907ea5d188e0cd141c82819
