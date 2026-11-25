# \#99687 [SC-Critical] Oracle rotation resets accepted feed timestamps and permits a forced liquidation

**Submitted on Nov 14th 2026 at 19:58:41 UTC by @Ailenia for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99687
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Impact:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.

## Summary

I can force-liquidate a near-maintenance trader immediately after a legitimate Oracle rotation by replaying a still-fresh Pyth component that the old Oracle already rejected. The rotation resets accepted feed timestamps. My PoC first proves the older component was harmless when it was accepted and when it was last admissible on the old Oracle. A small later price change makes it harmful, but only the replacement Oracle admits it. The neutral basket mark stays unchanged between attack and control, while the older component's higher confidence raises the adverse liquidation price and pays the keeper a 499.996725 USDG bounty from the trader's collateral.

## Root cause

I found that the live Oracle records each accepted Pyth component's publish time and rejects a component older than its last accepted value, but this protection disappears during the supported 72-hour Oracle rotation. `OrderLiquidationBatchLogic.applyOracleConfig` copies the active pricing policy without carrying forward the accepted component timestamps. `OrderRouter.applyOracleConfig` then installs the fresh Oracle with zero component watermarks. The next permissionless keeper can submit a still-fresh signed component older than one the old Oracle already accepted on-chain.

The neutral basket's aggregate price and publish time can remain identical in both reports, so the equal-time mark guard does not catch the rewind. Confidence is not bound to that equal-time guard. For a long position, `PletherOracle` adds 25% of aggregate confidence to the liquidation price. A higher-confidence older component can therefore turn a solvent position into a liquidatable one after rotation. The first faulty transition is the Router's installation of the replacement Oracle without preserving the active Oracle's accepted-feed state or enforcing a predecessor watermark.

## Reproduction

1. Fund Junior with a 10,000,000 USDG deposit and open the target's 1,000,000-unit long with 4,000.6 USDG margin through public V3 calls under the release risk settings. During an already proposed, mature Oracle rotation, track the EUR/USD move with four public mark updates spaced 15 minutes apart while the position remains solvent.
2. At `T-1`, the old Oracle accepts a complete higher-confidence report with EUR/USD at 1.17721370. A public liquidation reverts as solvent both then and at `T`, before the older component is superseded. At `T`, accept a lower-confidence second component at the same EUR/USD price. At `T+1`, accept another lower-confidence report with EUR/USD at 1.17721470, only about 0.0085 bp higher. The old Oracle rejects a public liquidation call with the higher-confidence `T-1` second component and current `T+1` other feeds with `PletherOracle__FeedTimeOutOfOrder`.
3. The owner finalizes the timelocked rotation. The new Oracle has a zero accepted timestamp for that component. From the same post-rotation state, the current report gives a 1.00109222 liquidation price and leaves the trader solvent. The mixed older report gives a 1.00109280 liquidation price, above the measured 1.00109251 maintenance threshold, and lets a permissionless keeper liquidate through `OrderRouter.executeLiquidation`. Both reports have the same neutral basket mark and basket timestamp.

The local Pyth mock supplies complete reports with the stated times and confidence values, and they meet the release freshness, divergence, and aggregate-confidence limits. Baseline confidence ratios are scaled from a checked-in six-feed Pyth update fixture. A separate checked-in signed Pyth fixture contains adjacent-second GBP/USD and USD/SEK updates with unchanged prices but higher older confidence; combining their intact older proofs with four newer feed proofs yields the same neutral basket mark and a 58-tick higher liquidation price under release weights. The Junior deposit uses public request, epoch-settlement, and claim calls. The owner performs documented release configuration and rotation; the adversarial report choice and liquidation are permissionless.

The economic PoC models the roughly 18.85-bp EUR/USD movement over one hour with accepted marks every 15 minutes; the final two-second step is about 0.0085 bp. This market path and a legitimate owner rotation are prerequisites; the keeper does not control them. The signed fork sample demonstrates the watermark reset with real Pyth proofs, but its sampled position is not close enough to maintenance to liquidate.

## Impact and accounting

My PoC starts both outcomes from one snapshot. In the control, the lower-confidence report cannot liquidate, and the trader closes through a public V3 order one second later with 8,100.027395 USDG left in the margin account. In the attack, liquidation at that same one-second-later execution time leaves 6,999.997362 USDG. The measured trader-balance difference between those complete branches is **1,100.030033 USDG**: **499.996725 USDG** credited to the keeper, **600.033308 USDG** added to Pool cash and Junior LP assets, and **0** to the treasury. This branch comparison includes the control close's charges and the one second elapsed in both branches; it is the exact outcome for those public paths, not a decomposition of the confidence shift alone. The PoC checks the Clearinghouse and Pool token movements against their accounted balances, unchanged token supply and LP shares, and no hidden trader claim. The keeper withdraws its bounty; the trader can withdraw only the residual balance. The forced liquidation cannot be reversed by a later fresh price update.

This bypasses the accepted-component monotonicity mitigation described in the published MED-7 response. The higher-confidence component was accepted on-chain while it was harmless, and public liquidation was still solvent immediately before the lower-confidence successor superseded it. The small later EUR/USD step first makes the harmful mixed report possible after the old Oracle's timestamp floor has already advanced. The old Oracle explicitly rejects that mix; only rotation makes it admissible. MED-13 concerns changed protection thresholds under a semantically different Oracle; my PoC rotates to an identical Oracle and targets liquidation. The published MED-18 equal-time mark fix also remains active: the two marks are equal, while confidence-dependent liquidation prices differ.

## Remediation

Before installing a replacement Oracle, carry forward the active per-feed accepted timestamps and accepted basket state for matching feeds, or enforce the active Oracle's accepted-feed timestamps as a Router-level floor across rotations. Reject a rotation that cannot preserve that floor while live positions can be liquidated. Bind equal-time confidence-dependent prices to a consistent accepted basket when deciding liquidation.

## Link to Proof of Concept

https://gist.github.com/niafreu/324af5f3f0d3f00941d181b5b30814a8

## Proof of Concept

## Step-by-step reproduction

1. Fund Junior with a 10,000,000 USDG deposit and open the target's 1,000,000-unit long with 4,000.6 USDG margin through public V3 calls under the release risk settings. During an already proposed, mature Oracle rotation, track the EUR/USD move with four public mark updates spaced 15 minutes apart while the position remains solvent.
2. At `T-1`, the old Oracle accepts a complete higher-confidence report with EUR/USD at 1.17721370. A public liquidation reverts as solvent both then and at `T`, before the older component is superseded. At `T`, accept a lower-confidence second component at the same EUR/USD price. At `T+1`, accept another lower-confidence report with EUR/USD at 1.17721470, only about 0.0085 bp higher. The old Oracle rejects a public liquidation call with the higher-confidence `T-1` second component and current `T+1` other feeds with `PletherOracle__FeedTimeOutOfOrder`.
3. The owner finalizes the timelocked rotation. The new Oracle has a zero accepted timestamp for that component. From the same post-rotation state, the current report gives a 1.00109222 liquidation price and leaves the trader solvent. The mixed older report gives a 1.00109280 liquidation price, above the measured 1.00109251 maintenance threshold, and lets a permissionless keeper liquidate through `OrderRouter.executeLiquidation`. Both reports have the same neutral basket mark and basket timestamp.

The local Pyth mock supplies complete reports with the stated times and confidence values, and they meet the release freshness, divergence, and aggregate-confidence limits. Baseline confidence ratios are scaled from a checked-in six-feed Pyth update fixture. A separate checked-in signed Pyth fixture contains adjacent-second GBP/USD and USD/SEK updates with unchanged prices but higher older confidence; combining their intact older proofs with four newer feed proofs yields the same neutral basket mark and a 58-tick higher liquidation price under release weights. The Junior deposit uses public request, epoch-settlement, and claim calls. The owner performs documented release configuration and rotation; the adversarial report choice and liquidation are permissionless.

The economic PoC models the roughly 18.85-bp EUR/USD movement over one hour with accepted marks every 15 minutes; the final two-second step is about 0.0085 bp. This market path and a legitimate owner rotation are prerequisites; the keeper does not control them. The signed fork sample demonstrates the watermark reset with real Pyth proofs, but its sampled position is not close enough to maintenance to liquidate.

## Runnable PoC
Secret PoC gist: https://gist.github.com/niafreu/324af5f3f0d3f00941d181b5b30814a8

The runnable Foundry test is in `OracleRotationFeedRewind.t.sol`. To reproduce from a clean directory with Foundry and Solc 0.8.35 available to Forge's offline compiler resolver, place the five gist files beside the pinned upstream checkout and run:

```sh
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether code
git -C code checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git -C code submodule update --init --recursive
mkdir -p findings/oracle-rotation-feed-rewind
git clone https://gist.github.com/324af5f3f0d3f00941d181b5b30814a8.git findings/oracle-rotation-feed-rewind/PoC
bash findings/oracle-rotation-feed-rewind/PoC/run.sh
```

The runner copies the test into the pinned Foundry checkout without changing production contracts. The test uses public Router, admin, order, liquidation, and Clearinghouse calls. Its only mock is the out-of-scope Pyth endpoint and settlement token supplied by the repository fixture.

For an additional signed-Pyth integration check on the Arbitrum Sepolia fork, run `bash findings/oracle-rotation-feed-rewind/PoC/run-fork.sh`. The runner uses the documented public RPC unless `ARB_SEPOLIA_RPC_URL` is set. Its two tests parse intact signed proofs with the deployed Pyth contract and show the old Oracle rejecting a regressed feed while the replacement accepts it through public Router calls. The accepted mark is unchanged, and the source-derived adverse price rises 58 ticks. This fixture leaves the position solvent in both branches, so the economic loss measurement above comes from the deterministic public-call PoC.

Expected result: the current report leaves the trader solvent, the older mixed report liquidates, and the test measures 499.996725 USDG keeper credit, 600.033308 USDG Pool/Junior gain, and 1,100.030033 USDG trader loss against the same-state control.
