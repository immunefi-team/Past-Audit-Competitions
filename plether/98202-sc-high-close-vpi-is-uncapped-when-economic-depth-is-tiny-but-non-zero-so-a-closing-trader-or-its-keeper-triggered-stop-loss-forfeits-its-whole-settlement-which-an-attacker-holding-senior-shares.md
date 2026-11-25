# \#98202 [SC-High] Close VPI is uncapped when economic depth is tiny but non-zero, so a closing trader (or its keeper-triggered stop-loss) forfeits its whole settlement, which an attacker holding Senior shares then redeems

**Submitted on Nov 5th 2026 at 22:44:12 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98202
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The close-side VPI charge is `k·(post² − pre²) / (2·D)`, where `D` is the pool's economic depth, `pool.totalAssets() − totalTraderClaims`. Only one case is guarded: `D == 0` creates no fresh charge (`CloseAccountingLib.sol:107`), and the docs state that rule for zero depth only. Nothing covers `D` between 1 atom and a few USDG, where the same formula yields charges of millions of USDG on an ordinary close.

That depth is reachable with public calls once the pool is in its documented claims regime (degraded mode, trader claims ≈ pool cash). A losing trader sizes a partial close so that its realized loss leaves `D` at a few USDG. From then on, every close that increases skew collects the closer's whole settlement:
- its realized profit;
- its position margin;
- free settlement that was never pledged to the position.

The shortfall is written off. A keeper-triggered stop-loss does the same to a trader who takes no action. The confiscated cash becomes LP equity and flows to the Senior tranche.

The PoC uses release parameters. An attacker holds the Senior shares and controls the depth-sizing trader, and pays 3.17 USDG in fees. It redeems **237,478.12 USDG** instead of **125,196.08 USDG** in the identical timeline without the bug, a gain of **+112,282.04 USDG**. The victim ends with **0** instead of **112,505.47 USDG**.

## Vulnerability Details

Line numbers refer to `immunefi-team/audit-comp-plether` at a82d4fe. The in-scope code is identical to the Arbitrum Sepolia v2.0.0 release.

### 1. The only guard is the exact-zero exemption

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:105-117
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

`CfdMath.calculateVPI` (`CfdMath.sol:150-161`) is `k·(post² − pre²)/(2·depth)` with no cap. The clamp that follows bounds only rebates; it never bounds charges. The router supplies `poolDepthAssets` as the net depth `pool.totalAssets() − totalTraderClaims`: `OrderRouterExecutionSidecar.sol:933/953` reads it and `CfdEnginePlanLib.sol:448-487` nets it.

The zero-depth rule was added in 6d6622f ("use net risk depth"). Its stated purpose is to "keep protective exits live" at exhausted depth. One atom above zero the exit is still live, but it confiscates the closer's funds.

### 2. The charge is collected from everything the account owns

The charge is an action charge. `MarginClearinghouse.sol:955-959` collects `min(charge, eligible)`, where `eligible` covers the spendable action reserve, free buying power and committed order margin, in addition to the closed position's pledge. `CfdEngineSettlementSidecar.sol:674` (`ActionChargeSettled`) records the rest as waived. The close executes and nothing reverts. The victim's realized profit is withheld against the charge, and its unrelated free settlement is taken.

### 3. Depth can be steered to a few USDG with public calls

In the claims regime, a winner's unpaid profit is booked as a trader claim, so `D = assets − claims` sits at 0. Any losing trader can choose `D` by closing part of its position: the realized loss enters pool assets. One coarse partial close gives `D ≈ 64 USDG`; two give `D ≈ 2.2 USDG`. The attacker's fees are 3.17 USDG. The attacker's trading loss is the same with or without the attack, as the control timeline shows.

### 4. The confiscated value becomes redeemable Senior equity

With claims fully covered, the victim's seized cash is free pool equity. Junior is first-loss and was already wiped in the regime, so the value restores impaired Senior principal. When the remaining positions close, the owner's documented recovery action `clearDegradedMode()` becomes possible: it requires cash ≥ max liability (`CfdEngine.sol:658-665`). WHITEPAPER.md §7.5 (lines 1335-1339) describes it as the protocol's recovery path: "After genuine restoration to A_eff ≥ L_max, the owner may clear the latch through the explicit recovery path". After it, the normal LP epoch settlement funds Senior redemptions, and the attacker's Senior shares redeem the victim's value.

`settleLpEpoch` reverts `HousePool__DegradedMode` until that point in both timelines. The owner action is therefore the protocol's ordinary exit path for every LP, not something the attack needs from a malicious owner. The program puts this case in scope: "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol".

### 5. The victim has no defense

- **Unbounded close:** the 5-argument `commitOrder` has no charge bound, so the victim's whole settlement goes.
- **Stop-loss:** the victim's stop-loss protection is triggered and executed by a public keeper. The program says "harm caused by any caller acting as a keeper is in scope". The result is the same total loss with no further action by the victim.
- **Bounded close:** a close with `maxActionChargeAssets` reverts `ConstraintViolation`. Withdrawals of positioned accounts revert `CfdEngine__DegradedMode` (`CfdEngineSettlementSidecar.sol:274`). The account cannot be liquidated, because it is solvent. Stored depth does not move by itself: 48 h later the charge is still 218,333 USDG, 8.4× the account's balance. The only exit costs everything.

## Impact Details

**Direct theft of user funds (Critical).** The PoC executes the theft end to end: the attacker realizes the victim's funds as USDG in its own wallet.

| Timeline (same snapshot) | Victim V closes | V ends with | Attacker's Senior redemption |
|---|---|---|---|
| Attack: V closes at D = 2.18 USDG | charge 2,819,144.31 USDG on a 100k close | **0** | **237,478.12 USDG** |
| Control: V closes after depth is restored | rebate 4.18 USDG | 112,505.47 USDG | 125,196.08 USDG |
| **Difference** | | **−112,505.47** | **+112,282.04** |

The attacker controls two accounts: the Senior LP and the depth-sizing trader L. The test asserts that L's trading account ends the same in both timelines (60,718.17 USDG, within 42 atoms of rounding), so the attacker's net gain across both accounts is **+112,282.04 USDG**. The third-party claimant W ends with exactly the same balance in both timelines. The taken value comes from the victim alone.

Cost to the attacker: 3.17 USDG in fees, about 1 : 35,000 against the amount taken. With the fix applied, the same test gives a net gain of 669.99 USDG instead of 112,282.04.

Every trader who closes in the skew-increasing direction while `D` is small loses the same way. Voluntary closes and keeper-triggered stop-losses are both affected. The attacker keeps `D` small because nothing else moves it in a degraded pool, where opens are blocked.

Other measured cases:

| Scenario | D before the close | Close charge | Victim settlement before → after |
|---|---|---|---|
| Control, before the claims regime | 1,011,121.71 | −27.22 (rebate) | 25,926.62 → 112,528.51 |
| One sized partial close | 63.90 | 96,884.59 | 25,926.62 → 15,616.92 (86,580 profit also forfeited) |
| Two sized partial closes | 2.18 | 2,819,144.31 (2,706,637.69 waived) | 25,926.62 → 0 |
| Stop-loss triggered by a keeper | 2.19 | 3,110,424.46 | 25,926.61 → 0 |
| Bounded close (max charge 1,000) | 2.19 | ConstraintViolation | no close, no withdraw, not liquidatable; charge 218,333 at t+48 h |

If the redemption step is not accepted as theft, the bounded-close case meets the High impact "Temporary freezing of funds for at least 24 hours". The total-loss cases are not griefing under the program's ratio rule: 3 USDG causing more than 100k USDG of damage.

### Precondition and feasibility (stated openly)

The pool must already be in the claims regime: trader claims ≈ pool cash, with degraded mode latched. Opens must leave assets ≥ max liability plus a settlement buffer (`CfdEnginePlanLib.sol:421` → `SolvencyAccountingLib.sol:93-104`), and LP redemptions reserve max liability plus claims (`HousePoolAccountingLib.sol:50-51`). So the regime needs one side's profit to be paid out and then the other side to win close to its maximum. In the PoC this is a 1.0 → 1.5 → 0.2 basket path with large positions on both sides.

The state is documented protocol behavior: degraded mode, trader claims and the zero-depth close rule. No trusted role and no oracle trust is needed. Under the program's Feasibility Limitations, "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it." The capital involved is well below the USD 100M threshold.

### Not a known issue

- **Documentation:** the docs state the zero-depth rule for zero depth only: ACCOUNTING_SPEC.md:665-669 ("A zero-depth close creates no fresh VPI charge or rebate"), SECURITY.md:1439-1440 and WHITEPAPER.md:866-867. SECURITY.md:1434-1445 lists the VPI limitations (zero depth, lifetime clamp, partial-close approximation, liquidation); small positive depth is not among them.
- **AI scan Info items:** "Temporary pool liquidity can reduce a skew-increasing close charge" and "... can amplify a skew-reducing close rebate" describe a trader moving `totalAssets` with its own liquidity to lower its own charge or raise its own rebate. This report is the opposite direction, with a different victim, a different defect and a different fix. The scan's remedy (commit-time or time-weighted depth) does not remove it: the depth here is genuinely tiny at both times.
- **Not "lack of liquidity":** the loss is not a delayed or unpaid payout. A formula singularity takes the trader's own collateral, unrelated to pool cash, and an attacker then redeems it.

## Recommendation

Treat depth below a floor as zero for the fresh close-side charge, or cap the fresh positive charge at a share of the closed slice's notional. The gist's `fix.diff` uses 5% of the closed notional. With it applied, the cliff tests fail and the repo's 178 close/VPI tests pass. Evaluating the VPI delta with `max(D, D_min)` is an equivalent option.

## Link to Proof of Concept

https://gist.github.com/the-red-ace/4395a2418911c1e0be32d484b6fa2c71

## Proof of Concept

## Proof of Concept

Gist: https://gist.github.com/the-red-ace/4395a2418911c1e0be32d484b6fa2c71. It contains `X2Cliff.t.sol`, `X2GridBase.sol`, `X2ErrNames.sol`, `fix.diff` and a README.

The tests run in the repo's own Foundry suite at a82d4fe with release risk parameters. They do not modify any in-scope contract. The only mocks are `MockPyth` price updates and mock USDG, both allowed by the program. All state changes go through `commitOrder`, `executeOrder`, `updateMarkPrice`, `createPositionProtection` and `triggerPositionProtection`, plus the owner's `clearDegradedMode()` recovery action before LP redemption. There is no `vm.store`, `deal` or `mockCall`.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/x2 && cp X2GridBase.sol X2ErrNames.sol X2Cliff.t.sol packages/perps/test/perps/x2/
forge test --root packages/perps --match-path test/perps/x2/X2Cliff.t.sol -vv
```

| Test | What it asserts |
|---|---|
| `test_X2C_Control_LargeDepth` | At normal depth the same close is an ordinary rebate, and V keeps 112,528.51. |
| `test_X2C_Cliff` | One sized partial close (D = 63.9): V ends below its pre-close settlement on a winning close. |
| `test_X2C_A_FineDepth_WholeSettlementConsumed` | D = 2.18: settlement goes to 0, the remainder is waived, nothing reverts. |
| `test_X2C_B_BoundedCloseFrozen` | ConstraintViolation, withdraw reverts DegradedMode, not liquidatable, depth unchanged after 48 h. |
| `test_X2C_C_AttackerIsJuniorLP` | Junior (first-loss, wiped) captures nothing; the Senior tranche rises by the victim's loss. |
| `test_X2C_D_ProtectionTriggeredCliff` | A keeper-executed stop-loss at D = 2.19 takes V's settlement to 0. |
| `test_X2C_E_SeniorAttackerRealizesVictimLoss` | Attack and control timelines from the same snapshot. The attacker's Senior redemption is 112,282.04 USDG higher and the victim 112,505.47 USDG poorer. The attacker's trading account and the third-party claimant end the same in both, so the net theft is 112,282.04 USDG. |
| `test_X2C_Escape_Liquidation_And_Wait` | V is not liquidatable, and seven days of waiting leave D unchanged. |

Expected output:
```
Suite result: ok. 8 passed; 0 failed; 0 skipped
test_X2C_E: attacker (Senior LP) redeems 237478122767 (attack) vs 125196083224 (control): +112282039543
            victim ends 0 (attack) vs 112505465699 (control); attacker's tuning fees 3169087
            attacker NET gain across both accounts: 112282039501
test_X2C_A: D before close 2178986 | vpiDelta 2819144309459 | V settlement 25926617619 -> 0
test_X2C_D: D at trigger 2186806 | charge 3110424456565 | V settlement 25926609799 -> 0
```

