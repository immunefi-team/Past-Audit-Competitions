# \#98762 [SC-Low] Commit-time open admission uses the unshifted mark, so admitted opens fail at an unchanged price and forfeit their bounty

**Submitted on Nov 10th 2026 at 02:31:02 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98762
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

**The promise.** The planner states that too-small and initial-margin failures are "commit-time rejectable" (`CfdEnginePlanLib.sol:73-74`). The Router implements this: at commit, `_validatedOpenBounty` checks the minimum notional and runs the full planner preview `previewOpenRevertCode`, and it rejects predictably invalid opens before any bounty is reserved.

**The mismatch.** Both checks use the **unshifted commit mark**. At execution the Oracle prices an open at the **adverse price**, which is the mark moved against the trader by `confidence * adverseConfidenceMultiplierBps / 10_000` (`PletherOracle.sol:905-921`): down for a LONG, up for a SHORT. The planner then re-runs at that price. At release parameters, the engine's size floor (`bountyBps = 10`, `minBountyAssets = 1e6`) is exactly the Router's 1,000 USDG minimum. Initial margin is likewise checked at the shifted price. Nothing gives the commit check any headroom for the shift.

**The consequence.** Orders the Router admitted fail at execution **with no price move at all**. They fail as `UserInvalid` (`PlannerRejected`), and per policy "ordinary open failure pays the executor from its reserved bounty" (`SECURITY.md:728`). The trader pays the keeper for an order that could never fill and that the commit check was meant to reject.

**Measured in the PoC** (release Router config including `adverseConfidenceMultiplierBps = 2,500`, mark 1.00 unchanged at execution):
- **LONG, 10 lots = 1,000 USDG, the advertised minimum.** Pyth confidence 1 bp or 5 bps: terminal failure, the keeper receives 0.1 USDG and the trader loses 0.1 USDG. With confidence 0, or with 11 lots, the order opens.
- **SHORT, 20 lots = 2,000 USDG, at the smallest margin the commit preview accepts** (what a UI quotes as maximum leverage, 32.801 USDG). Confidence 5 bps: terminal failure with `INSUFFICIENT_INITIAL_MARGIN`, and the keeper receives the 0.2 USDG bounty cap. With confidence 0 the order opens, and the LONG mirror at its own minimum margin opens.

## Vulnerability Details

### 1. Commit-time admission evaluates the mark

```solidity
// packages/perps/src/router/OrderLiquidationBatchLogic.sol:412-446
uint256 commitPrice = _commitReferencePrice(engine);   // engine.lastMarkPrice(), unshifted
uint256 lots = sizeDelta / CfdTypes.SIZE_QUANTUM;
...
uint256 notionalAssets = lots * commitPrice;
if (notionalAssets < host.minOpenNotionalAssets()) {
    revert OrderRouter__CommitValidation(11);
}
...
uint8 revertCode =
    host.engineLensForCommit().previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice);
...
        if (OrderFailurePolicyLib.isPredictablyInvalidOpen(failureCategory)) {
            revert OrderRouter__PredictableOpenInvalid(revertCode);
        }
```

### 2. Execution evaluates the adverse-shifted price

```solidity
// packages/perps/src/PletherOracle.sol:905-921
function _adverseOrderPrice(OrderExecutionRequest calldata request, uint256 price, uint256 confidence)
    internal view returns (uint256) {
    uint256 shift = (confidence * adverseConfidenceMultiplierBps) / 10_000;
    ...
    bool adverseUp = request.side == CfdTypes.Side.SHORT ? !request.isClose : request.isClose;
    if (adverseUp) {
        return price + shift;
    }
    ...
    return price - shift;
}
```

### 3. The engine floors are evaluated at that price

```solidity
// packages/perps/src/libraries/OpenAccountingLib.sol:142-148
uint256 resultingNotionalAssets = newLots * inputs.price;
...
rawLiquidationRequirementAssets = (resultingNotionalAssets * inputs.riskParams.bountyBps) / 10_000;

// packages/perps/src/libraries/CfdEnginePlanLib.sol:497-500
if (rawLiquidationRequirementAssets < snap.riskParams.minBountyAssets) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.POSITION_TOO_SMALL;
}
```

With release `bountyBps = 10` and `minBountyAssets = 1e6` (`DEPLOYMENT.md:138-139`), the floor is exactly 1,000 USDG of notional, the same as `minOpenNotionalAssets = 1_000e6` (`DEPLOYMENT.md:148`). It is measured at the execution price, which is lower for a LONG. In the same way, the SHORT's notional, fee and initial-margin requirement rise with the upward shift, and the planner returns `INSUFFICIENT_INITIAL_MARGIN`.

### 4. The failure is charged to the trader

`POSITION_TOO_SMALL` and `INSUFFICIENT_INITIAL_MARGIN` are classified as `UserInvalid` at execution (`CfdEnginePlanLib.sol:112-128`) and settled as terminal `PlannerRejected`. The Router pays the reserved open bounty (1 bp of notional, minimum 0.01 and maximum 0.2 USDG) to the keeper (`SECURITY.md:728`, `ACCOUNTING_SPEC.md:997`). The same codes are listed as commit-time rejectable (`CfdEnginePlanLib.sol:73-92`), and the Router does run the preview for exactly that purpose, but at a price that cannot see the failure.

### Relation to prior audits and known issues

- **Cecuro LOW-31 (Won't fix)** covers undersized opens losing a bounty **before the first mark update**, when there is no usable commit mark.
- **Cecuro LOW-57, MED-16** and the known limitation "stale cached marks can understate opening notional and execution bounties at commitment" (`SECURITY.md:1349-1352`) cover a **stale** commit mark and price moves between commit and execution.
- In this report the commit mark is **fresh and unchanged** at execution. The failure comes from the Oracle's own confidence shift being applied on top of floors that the commit check evaluates without it. The fix is in the commit check itself.

## Impact Details

**Low, "Contract fails to deliver promised returns, but doesn't lose value".** The Router admits orders that it promises to reject at commit, and these orders deterministically cannot fill.

| Order (mark 1.00, unchanged) | Pyth conf | Result | Keeper | Trader |
|---|---|---|---|---|
| LONG 10 lots (1,000 USDG) | 5 bps | Terminal, `PlannerRejected` | +0.1 USDG | −0.1 USDG |
| LONG 10 lots | 1 bp | Terminal, `PlannerRejected` | +0.1 USDG | −0.1 USDG |
| LONG 10 lots (control) | 0 | Opens | — | — |
| LONG 11 lots (control) | 5 bps | Opens | — | — |
| SHORT 20 lots, minimum margin at the mark | 5 bps | Terminal, `INSUFFICIENT_INITIAL_MARGIN` | +0.2 USDG | −0.2 USDG |
| SHORT 20 lots, same margin (control) | 0 | Opens | — | — |
| LONG 20 lots, its own minimum margin (control) | 5 bps | Opens | — | — |

- **Who is hit.**
  - LONG opens whose notional sits within the confidence shift above 1,000 USDG. One lot is 100 tokens (about 100 USDG), so this is about 0.25 % of exact-minimum LONG orders at the 10 bps confidence cap, and about 0.03-0.08 % at typical FX confidence.
  - SHORT opens of **any size** placed at the maximum leverage that the commit preview accepts. Pyth confidence is never zero in practice, so these fail deterministically.
- **Loss per event:** the reserved open bounty, 0.01-0.2 USDG. Margin is refunded and no protocol funds are lost.
- **Keepers** can pick these orders out of the queue as guaranteed bounty income with no position risk.

### Preconditions

- Release parameters (as shipped): floors equal at 1,000 USDG, adverse multiplier 2,500, confidence cap 10 bps.
- Nonzero Pyth confidence at execution, which is always the case.
- An open at the advertised minimum size (LONG) or at the maximum leverage the commit preview accepts (SHORT).

## Recommendation

Run the commit-time admission at the **worst execution price** an open can receive, which is the mark shifted by `basketMaxConfidenceRatioBps * adverseConfidenceMultiplierBps`. Apply it to both the minimum-notional check and the planner preview:

```diff
-        uint256 notionalAssets = lots * commitPrice;
+        uint256 worstPrice = _worstOpenPrice(host, side, commitPrice);
+        uint256 notionalAssets = lots * (worstPrice < commitPrice ? worstPrice : commitPrice);
         if (notionalAssets < host.minOpenNotionalAssets()) {
             revert OrderRouter__CommitValidation(11);
         }
+        commitPrice = worstPrice;
```

`_worstOpenPrice` returns `price ∓ price * basketMaxConfidenceRatioBps * adverseConfidenceMultiplierBps / 1e8` (full diff in the gist, `fix.diff`).

With the fix, both PoC orders are rejected at commit, before any bounty is reserved: the LONG with `CommitValidation(11)` and the SHORT with `PredictableOpenInvalid(6)`.

The fix is a stricter admission policy, and existing tests notice it. In the trader, keeper and regression suites (603 tests), 11 tests fail with the fix and 0 fail without it:
- 10 of them commit exactly at a boundary (the exact minimum notional, the exact margin or the exact cap) that the fix now rejects at commit;
- 1 asserts the bounty on the unshifted notional (111,977 vs 112,000).

These tests encode the current admission at the mark, which is the behaviour this report questions. They need new boundary values, not a code change.

**Deployment note (EIP-170).** At a82d4fe, `OrderRouterLiquidationBatchSidecar` has only 30 bytes of runtime margin, and this change adds about 780 bytes. The diff demonstrates the logic. For deployment, the worst-price helper has to live in a contract with room, for example as a Router or Oracle view that the sidecar calls once (the Oracle has about 7 KB of margin).

An alternative is to classify an execution-time failure of a commit-rejectable code on an order that passed the commit preview as a protocol-state invalidation, and refund the bounty.


## Link to Proof of Concept

https://gist.github.com/the-red-ace/22284ed3db6ac85dfc8d27e9f8dd666e

## Proof of Concept

## Proof of Concept

**Gist:** https://gist.github.com/the-red-ace/22284ed3db6ac85dfc8d27e9f8dd666e. It contains the test file, `fix.diff` and a README with the exact folder.

**How the tests run:** in the repo's own Foundry suite at a82d4fe, on `BasePerpTest`. Production Router, Engine, Clearinghouse and `PletherOracle` are used, with `MockPyth` for Pyth.
- The release Router config is set in the test: minimum notional 1,000 USDG, open bounty 1 bp (0.01-0.2 USDG) and `adverseConfidenceMultiplierBps = 2,500`. The test asserts that the Oracle reads 2,500.
- Risk parameters are the fixture defaults, which equal release (`bountyBps = 10`, `minBountyAssets = 1e6`).
- Cheats: `vm.warp`/`vm.roll`, MockPyth price and confidence, USDG mint, snapshots. No in-scope contract is modified.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
cp CommitPreviewBasis.t.sol packages/perps/test/perps/spec/oracle/
forge test --root packages/perps --match-contract CommitPreviewBasisTest -vv
```

| Test | What it shows |
|---|---|
| `test_LongAtMinimum_unchangedMark_confidenceShiftOnly` | A 10-lot LONG at mark 1.00, executed at the same mark with confidence 5 bps and then 1 bp: no position, status 3, reason 7, keeper +0.1 USDG and trader −0.1 USDG (asserted). Controls: confidence 0 opens; 11 lots at 5 bps opens. |
| `test_ShortAtQuotedMaxLeverage_unchangedMark_confidenceShiftOnly` | A 20-lot SHORT at the smallest margin the preview accepts at the mark. The preview returns 0 at the mark and 6 at the shifted price (asserted). Execution fails terminally and the keeper takes 0.2 USDG. Controls: confidence 0 opens; the LONG mirror opens. |

Expected output (reproduced on a clean a82d4fe checkout with only the gist file):
```
[PASS] test_LongAtMinimum_unchangedMark_confidenceShiftOnly()
[PASS] test_ShortAtQuotedMaxLeverage_unchangedMark_confidenceShiftOnly()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

With `fix.diff` applied (`git apply fix.diff` from the repo root), both tests fail on purpose at the `commitOrder` call: `OrderRouter__CommitValidation(11)` for the LONG and `OrderRouter__PredictableOpenInvalid(6)` for the SHORT. The orders are rejected before any bounty is reserved.
