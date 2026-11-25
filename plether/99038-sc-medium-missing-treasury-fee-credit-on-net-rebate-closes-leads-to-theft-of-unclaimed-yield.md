# \#99038 [SC-Medium] Missing treasury fee credit on net-rebate closes leads to theft of unclaimed yield

**Submitted on Nov 11th 2026 at 15:46:48 UTC by @Martelle88622 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99038
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

The pool is the counterparty for every trade. Pool holders fund it. Traders post margin and open positions. A position is a long or short bet with size and margin.

Each trade pays a small trading fee. The fee is 4 in 10,000 of trade size. So a 200,000 dollar trade pays 80 dollars. The fee belongs to one account called the treasury.

There is also an imbalance charge. When a trade makes pool imbalance worse, the trader pays extra. That cash goes to the pool. When a trade makes imbalance better, the trader gets a refund. That refund comes from pool cash. Below I call it the imbalance refund. It can be larger than the fee.

The bug: when the imbalance refund is larger than the fee, the treasury gets zero. The trader still pays the fee because it is taken from the refund. The pool keeps that fee as its own cash. The in-scope engine lens preview also shows zero fee, though its field is documented as the assessed fee. A consumer that copies that zero into its fee bound will see its close fail and stay failed. The position stays open.

The proof runs at release settings. The imbalance factor is 0.01, the skew cap is 0.4, and the support cost rate is 500. Pool size is 1,000,000 dollars. Close size is 200,000 dollars. Fee is 80 dollars. Refund is about 199.96 dollars. The trader gets about refund minus 80 dollars. The treasury gets zero. The pool keeps the 80 dollars. Exact payout is read from the preview, since live depth and support cost move it by dust.

## Vulnerability Details

First some terms, so a new reader can follow:

- A close ends a position in full.
- The assessed fee is the fee worked out from size. For a 200,000 dollar close it is 80 dollars.
- The planner is the code that plans a close before it runs.
- The preview is a read only view of that plan. Wallets use it to show numbers and to set limits.
- A bound is a user limit. The fee bound is the most the user allows for fees. If the real fee is above the bound, the close fails and stays failed. The position stays open.
- The receipt is the stored result of a run. It records what the run charged.
- Amounts use 6 decimals. So 1,000,000 means 1 dollar. Prices use 8 decimals.

Here is what is wrong, step by step.

1. The planner builds one signed net number for the close. It adds the imbalance refund, the fee, and carry. Carry is a small support cost. If that net is below zero, the code records one refund for the whole net. It records no separate charge. This is `CfdEnginePlanLib.sol:1041-1060`. Lines 1059 to 1060 are the refund branch.
2. The planner then sets the fee it will send to the treasury. It only allows fee from a positive charge. The math is at `CfdEnginePlanLib.sol:1063-1073`. With a net refund there is no positive charge. So the allowed amount is zero. The planned fee is set to zero. The assessed fee is still 80 dollars.
3. Settlement pays the refund from pool cash. It tops up the treasury only from the planned fee. The top up call is at `CfdEngineSettlementSidecar.sol:626-628`. The planned fee is zero, so the top up is zero. The 80 dollars taken from the trader stays in the pool.
4. The open path does not have this bug. An open keeps the full assessed fee at `CfdEnginePlanLib.sol:399`. It sends it to the treasury at `CfdEngineSettlementSidecar.sol:498`. A close with a positive net charge also sends it. The proof test checks both controls. Each control sends the full 80 dollars to the treasury.
5. The engine lens preview copies the planned zero. This is `CfdEngineLens.sol:399`. Its field is documented as the fee assessed on the close, at `ICfdEngineTypes.sol:268`. So this in-scope preview shows zero fee on a close where 80 dollars was assessed and taken. The separate commitment-aware preview used by the documented bound-derivation flow reports the assessed fee correctly; the proof test checks this as a negative control. Only the engine lens understates it.
6. The run check counts the real fee. This is `CfdOrderPolicyEvaluatorBase.sol:416`. It sets the checked fee total to assessed fee. The assessed fee is also stored in the run record at `:431`. The receipt copies it at `OrderRouterExecutionSidecar.sol:791`. So the receipt shows 80 dollars while the treasury got zero. The proof test checks this on the executed close itself.
7. The fee bound check enforces the real fee. This is `CfdOrderPolicyEvaluatorBase.sol:514-515`. It rejects the close if the checked fee total is above the user fee bound. A bound copied from the engine lens preview is zero. The real total is 80 dollars. So the close fails and stays failed. The position stays fully open.

The fee math comes from `CloseAccountingLib.sol:121-123`. It works out the 80 dollar fee from size.

## Impact Details

- Each net refund close diverts its fee. The fee is taken from the trader and left in the pool. The proof test shows the treasury gain is zero. Pool cash falls by refund minus fee only. No later step moves the fee to the treasury.
- The loss repeats with volume. Each such close diverts one more fee. A trader who also holds pool shares can steer the fee to the pool and later withdraw a share of it. This is a lasting move of protocol income, not a timing gap.
- The wrong engine lens preview breaks exits built from it. The proof test reads the lens fee of zero. It submits the same close twice from one saved state. The run with a fee bound of 80 dollars ends flat. The run with a fee bound of zero fails on the fee bound with actual 80 and limit zero. The position stays at full size. A negative control in the same test shows the documented `CfdClosePreview` flow reporting the full 80 dollars, so the failed exit is scoped to consumers of the in-scope engine lens field, which is documented as the assessed fee.
- No user principal is taken in the base fee path. The trader net matches the refund minus fee math. The harm is the fee destination, the wrong lens preview, and the failed exit built from that preview.

## Why this is not expected behavior

- The fee belongs to the treasury when it can be paid in cash. Here it is paid in cash from the refund. The pool holds the cash. The open path and the charged close path both send it.
- The rule for cash first top ups is at `ACCOUNTING_SPEC.md:781-782`. The rule that only cash collected fees and free cash funded top ups become treasury margin is at `ACCOUNTING_SPEC.md:1118`.
- The lens preview field is the assessed fee, not the paid fee. It shows zero while the run charges 80 dollars.

## Limits and open points

- Every run uses release settings, pinned and asserted in the test itself: imbalance factor 0.01, skew cap 0.4, support cost rate 500, maintenance margin 0.1%, initial margin 0.2%, fee 4 in 10,000. The only test choice is pool depth of 1,000,000 dollars with a 200,000 dollar close. That yields a refund of about 199.96 dollars against the 80 dollar fee. Exact trader gain is read from the preview, within 1,000 units, since live depth and support cost move it by dust.
- This report claims no direct loss of user principal in the base path. The severity claim rests on the lasting move of assessed protocol income; the lens-induced failed exit is a secondary impact scoped to engine-lens consumers, as the negative control above shows.
- No owner, keeper, or governance step is needed. All proof runs are normal public orders.

## References

- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1060` (signed net, refund branch), `:1063-1073` (fee allowed only from positive charge), `:399` (open keeps assessed fee)
- `packages/perps/src/CfdEngineSettlementSidecar.sol:498` (open sends fee), `:602-628` (close pays refund and tops up treasury from planned fee only)
- `packages/perps/src/CfdEngineLens.sol:399` (preview fee field)
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:416, 431, 514-515` (run check counts assessed fee, stores it, enforces fee bound)
- `packages/perps/src/OrderRouterExecutionSidecar.sol:791` (receipt copies assessed fee)
- `packages/perps/src/libraries/CloseAccountingLib.sol:121` (assessed fee from notional)
- `packages/perps/DEPLOYMENT.md:130-145` (release imbalance factor 0.01, skew cap 0.4, support cost 500, margins 0.1%/0.2%, fee 4)
- `packages/perps/src/interfaces/ICfdEngineTypes.sol:268` (lens fee field documented as the assessed fee)
- `packages/perps/CLOSE_PREVIEW.md:52, 88` (bounds derived from fresh assessments; zero maximum bounds are real zero allowances)
- `packages/perps/ACCOUNTING_SPEC.md:781-782` (fee top ups follow payout order), `:1118` (only cash collected fees and free cash top ups become treasury margin)

## Link to Proof of Concept

https://gist.github.com/a-sklyarov2/62f6846cfb84dd1da6e08e07c9001188

## Proof of Concept

## Proof of Concept

Runnable Foundry test at pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No in scope changes. Mocks only for price updates and the USDG token. One file, included separately.

To run, place the file at `packages/perps/test/perps/PocCloseNetting.t.sol` and run:

```bash
forge test --root packages/perps --match-path test/perps/PocCloseNetting.t.sol -vv
# [PASS] test_NetRebateCloseWithholdsTreasuryFee()
# [PASS] test_LensDerivedFeeBoundMakesNetRebateCloseFail()
```

The tests use only public steps. A trader calls `OrderRouter.commitOrder`. An unrelated executor calls `OrderRouter.executeOrder`. The mark refresh calls `OrderRouter.updateMarkPrice`. No protocol contract is faked. No stored value is changed by hand. No engine call is made directly. No owner or rescue step is used. The release configuration is pinned in the test and asserted at runtime.

Test 1 builds a one sided short book. It heals it with a long. It restores the short book with a charged close. Then it closes the short with a larger refund. It checks the trader gets the previewed net payout within 1,000 units. It checks the treasury gain is zero. It checks pool cash falls by the same net payout. It checks the authenticated execution receipt records the full 80 dollar fee. It also checks the preview shows zero fee. And it checks that both control paths send the full 80 dollars to the treasury.

Test 2 reads the engine lens fee of zero. A negative control first shows the documented `CfdClosePreview` flow reporting the full 80 dollar fee for the same close. It then submits the same close twice from one saved state. The run with a fee bound of 80 dollars ends flat. The run with a lens-derived fee bound of zero fails on the fee bound with actual 80 and limit zero. The position stays at full size.

No setup beyond the repo checkout is needed. The file brings its own trader, healer, and executor accounts.
