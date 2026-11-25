# \#99268 [SC-Critical] Refundable VPI charge inflates Junior NAV and lets an LP-trader drain other LPs

**Submitted on Nov 12th 2026 at 19:03:53 UTC by @zcai for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99268
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief
HousePool counts a positive open-position VPI charge as final pool revenue when it prices Junior redemptions. The same position can later reclaim that charge when it closes and restores skew. A Junior LP that opens near the skew cap immediately before its redemption settles can redeem at an inflated price, close the position, and receive most of the same charge back. The attacker profits at the expense of the remaining Junior LPs.

The attack repeats every three hours with one pool of capital. In a six-cycle test at the release parameters, a 2.9 million USDG Junior stake earned 7,471.72 USDG relative to a no-attack control. The honest 1 million USDG Junior LP lost 12,113.38 USDG and ended below its original principal. The same test remained profitable at the protocol's 10 basis point confidence cap.

## Vulnerability details

### Root cause
When an open increases skew, CfdEngineSettlementSidecar sends the positive trade cost, net of protocol fees, to HousePool through recordClaimantInflow() as Revenue with CashArrived (CfdEngineSettlementSidecar.sol:446-463). The next reconcile routes that revenue through the waterfall. It first restores seniorPrincipal to its high-water mark and then increases juniorPrincipal (HousePoolReconcilePlanLib.sol:48-104; HousePoolWaterfallAccountingLib.sol:140-159).

The pool's LP-pricing snapshot does not subtract the signed vpiAccrued of open positions (HousePoolAccountingLib.sol:60-78). _settleLpEpoch() reconciles before it prices Junior redemptions and deposit activations (HousePool.sol:824-939). The live LP settlement fee is zero (HousePool.sol:1147-1155). A Junior redemption that settles while the charge is open therefore receives a share of a charge that is still refundable.

On close, CloseAccountingLib clamps the VPI delta only so that proportionalAccrualAssets + vpiDeltaAssets >= 0 (CloseAccountingLib.sol:103-118). When the position returns skew toward zero, the negative VPI delta can reclaim the full accrued charge. CfdEnginePlanLib converts the negative action net into actionRebateAssets, which HousePool pays in cash (CfdEnginePlanLib.sol:953-978, 1040-1061).

### Attack path
1. Before the five-minute cutoff, the attacker requests redemption of its Junior shares.
2. About two minutes before the epoch boundary, the attacker commits and executes an open that moves skew close to the 0.4 cap. The position pays a positive VPI charge into HousePool.
3. At the epoch boundary, the attacker calls the permissionless LP settlement. Reconcile adds the charge to Junior principal before the redemption price is calculated.
4. The attacker claims the inflated redemption and closes the position. Closing restores skew and pays the refundable charge back, less the close fee.
5. The attacker re-deposits the fixed stake, waits through the cooldown, and repeats the cycle.

### Difference from known issue SCAN-31
SCAN-31 concerns a reduced close-time depth denominator amplifying a skew-reducing close rebate. This report has a different root cause and remains profitable without that amplification. The close rebate here is bounded by the VPI clamp at the original charge minus the close fee. The profit comes from pricing a redemption with a refundable open-position charge in Junior NAV. A redemption price that excludes the charge removes the profit.

## Impact details
The PoC demonstrates a repeatable transfer of existing Junior LP principal to the attacker. It is profitable at a flat oracle price, so the profit comes from the accounting error rather than a favorable market move.

In the main six-cycle run at one basis point confidence:
- attacker Junior stake: 2.9 million USDG, about 74.4% of Junior capital
- average attacker profit: about 1,245 USDG per cycle
- cumulative attacker profit: 7,471.72 USDG
- cumulative honest-LP loss: 12,113.38 USDG
- honest LP final marked value: 987,801.27 USDG from a 1 million USDG deposit
- protocol fees: 4,653.76 USDG
- minimum cycle time: three hours

The loss is unbounded over repeated cycles. The attacker's fixed re-deposit buys slightly more shares after each cycle because the other LPs have been diluted, so re-entry does not consume the gain.

At the protocol's 10 basis point confidence cap, the same six cycles produced 4,171.46 USDG of attacker profit and an 8,816.69 USDG honest-LP loss.

The base case becomes profitable between about 41% and 50% of Junior ownership. With a third party already holding skew near 0.3 of pool depth, an attacker holding 40% of Junior earned about 147.50 USDG per cycle for six cycles. The measured break-even stake in that configuration is about 24% to 26% of Junior capital.

### Capital and financial risk
The required investment does not invalidate the attack under this competition's feasibility rules. The base attack needs a dominant Junior position. On the tested deployed configuration, that is tens of millions of USDG, below the program's USD 100 million case-by-case threshold. Pre-existing third-party skew reduces the required Junior share.

The attacker bears directional price risk during the open-position window. The PoC quantifies it:
- a flat basket price produces about 1,245 USDG profit per cycle
- a five basis point adverse move leaves about 272 USDG profit
- the measured break-even adverse move is about 6.4 basis points
- a 20 basis point adverse move loses about 2,643 USDG

The window between settlement, claim, and close can be compressed to roughly one block. The expected gain remains positive over repeated flat or small-move cycles, but the attacker bears tail risk. Under the program's published feasibility rule, financial risk supports a downgrade only when it massively outweighs the reward. Here the risk is quantified and does not eliminate the repeatable positive expected value.

## Recommendation
Exclude the signed sum of open-position vpiAccrued from distributable LP equity when HousePool prices deposits and redemptions. Release each position's amount only when the position closes or is liquidated. This keeps refundable charges and locked rebates out of LP prices until the obligation settles.

## References
- Scope commit: a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Affected asset: packages/perps/src/HousePool.sol
- Other relevant code: CfdEngineSettlementSidecar.sol, HousePoolReconcilePlanLib.sol, HousePoolWaterfallAccountingLib.sol, HousePoolAccountingLib.sol, CloseAccountingLib.sol, and CfdEnginePlanLib.sol


## Link to Proof of Concept

https://gist.github.com/i-am-zcai/b281a2eb26fa2178e0323921120a5ca3

## Proof of Concept

## Proof of concept

Secret Gist: [Plether 771a9be6 runnable PoC](https://gist.github.com/i-am-zcai/b281a2eb26fa2178e0323921120a5ca3)

The linked secret Gist contains the runnable Foundry test, the patch, and the complete November 12 verification output. It runs against the exact scope commit and uses the project's BasePerpTest stack with the real Engine, settlement sidecar, Clearinghouse, HousePool, both TrancheVaults, PletherOracle, and OrderRouter. MockPyth replaces the out-of-scope signed-price component, as the competition rules permit.

The test covers six consecutive attack cycles, one and 10 basis point confidence settings, several attacker ownership bands, standing third-party skew, favorable and adverse basket moves, and the exact minimum cycle time.

### Steps
1. Check out commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
2. Apply 771a9be6-repeat.patch from the secret Gist.
3. Run:

```sh
cd packages/perps
FOUNDRY_VIA_IR=false forge test --offline --fuzz-runs 1 \
  --match-path 'test/perps/regression/OpenVpiChargeJuniorExitRepeatCycle.t.sol' -vv
```

Expected result:

```text
Suite result: ok. 8 passed; 0 failed; 0 skipped
```

The November 12 verification run produced eight passes and zero failures.

### Key output
```text
cumulative attacker profit vs control: 7471720322
honest Junior value drop vs control:    12113381209
per-cycle average profit:               1245286720
elapsed seconds:                        64800
```

Values are six-decimal USDG atoms. The test compares each attack with a no-trade control from the same EVM snapshot and performs a real redemption of half the honest LP's shares at the end to confirm the marked loss is realizable.

The PoC uses a local deployment from the frozen scope commit and does not test against a live network.
