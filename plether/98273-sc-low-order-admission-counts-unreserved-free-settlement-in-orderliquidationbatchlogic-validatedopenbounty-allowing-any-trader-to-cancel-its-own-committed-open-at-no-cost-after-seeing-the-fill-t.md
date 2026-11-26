# \#98273 [SC-Low] Order admission counts unreserved free settlement in OrderLiquidationBatchLogic._validatedOpenBounty, allowing any trader to cancel its own committed open at no cost after seeing the fill tick, so Junior LPs absorb only the fills that favour the trader

**Submitted on Nov 6th 2026 at 15:51:34 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98273
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Any trader can turn a committed skew-reducing open into a free option against the Junior LPs. Order admission (`OrderLiquidationBatchLogic._validatedOpenBounty`) approves the open while counting the account's free settlement, but nothing reserves that free settlement for the order. After the order's first post-commit Pyth tick is known, the trader decides. If the market moves its way, it executes the order at that tick and profits. If the market moves against it, it withdraws the free cash and executes the order itself. The withdrawal is allowed because the account is still flat. The order then fails with `PlannerRejected`, and the reserved bounty comes back to the trader. The kill costs the trader nothing, and the Junior LPs never collect the loss that the filled order would have paid.

At the release risk parameters, take a 100,000 USDG order and a 2% move. The Junior LP ends 200.000001 USDG below a twin run where the same order fills, and the trader avoids a 279.200001 USDG loss. With a 390,000 USDG order at a 5% move, the Junior gap is 780.000001 USDG. In the favourable branch the trader keeps +1,913.130026 USDG.

## Vulnerability Details

All cites are at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, under `packages/perps/src`.

**1. Admission counts free settlement.** `_validatedOpenBounty` (`router/OrderLiquidationBatchLogic.sol:404`) calls `previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice)` against the account's current state. If the preview predicts failure, the commit reverts (`router/OrderLiquidationBatchLogic.sol:431-445`). The only thing the commit reserves is the execution bounty (`:448-454`). The preview's open plan draws the VPI rebate reserve from free settlement first, and only the remainder from the order's new pledge. If the pledge cannot cover that remainder, the open fails with `VPI_REBATE_RESERVE_UNFUNDED` (`libraries/CfdEnginePlanLib.sol:561-571`). A skew-reducing open whose margin sits exactly at the admission bar is therefore admitted only because free cash covers its VPI reserve.

**2. Nothing holds that free settlement until execution.** For an account with no position, `validateWithdraw` checks only the settlement floor, and only when one is set. Otherwise it returns (`CfdEngineSettlementSidecar.sol:260-272`). The trader can withdraw all of `freeSettlementAssets` while its own open is pending. The test does exactly that.

**3. Execution re-plans on the live balance.** At execution, free settlement is recomputed from the current balance (`libraries/MarginClearinghouseAccountingLib.sol:153-162`). The admitted open now fails in the planner. The test asserts `TerminalReason.PlannerRejected` (value 7, `OrderV3Types.sol:85-97`). The log line is `kill terminal reason 7`.

**4. Self-execution refunds the bounty.** When the bounty recipient is the order's own account, the reserved bounty goes back to it (`OrderRouter.sol:336-337`). The kill costs the trader zero. The test asserts that the trader's balance after the kill equals its balance at commit.

The implementation has a gap: admission relies on free settlement that nothing reserves between the fill tick and execution. The project states the property this breaks. `packages/perps/SECURITY.md:647` says live execution "settles against the first unique Pyth tick strictly after the recorded `commitTime`". `SECURITY.md:658-659` says:

> "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options,"

A withdrawal followed by self-execution gives the trader a cancellation path that it can use after it knows the tick. The commit binds the trader only when the market moves its way.

How this report differs from prior items:
- **SC-M4** ("Risk Free Option in 60Sec Oracle Window", SC Audit Studio pre-audit, marked fixed). That issue relied on keeper tick selection and a `SOLVENCY_EXCEEDED` refund triggered by an LP withdrawal. First-tick binding fixed it, and this path uses the bound first tick. Here the trader fails its own order by withdrawing its own free cash, so the option survives the fix.
- **Known scan item "Order execution accepts oracle updates older than the engine's cached mark"** (known-issues id 1567). It covers filling at an older favourable tick, which is the keep branch. This report covers the costless kill of the adverse branch.
- **Cecuro HIGH-1** (Arbitrum/Pyth clock alignment). It has a different root.

## Impact Details

The trader takes the favourable fills and refuses the adverse ones at no cost. The Junior LP keeps the exposure that the order would have offset and pays in every branch. Selected impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield. The asset taken is Junior LP principal.

Parties, all separate accounts:
- the trader `TRADER` (0xA11CE, no role)
- the counterparty `PEER` (0xB0B), which holds the long book that the trader's short would offset
- the Junior LP: the test contract, which holds the harness's 1,000,000 USDG Junior deposit plus the 1,000 USDG seed

Each branch starts from the same `vm.snapshotState` taken after the commit. All figures below come from `forge-F74Release.log`, run at the release risk parameters in `DEPLOYMENT.md:131-141`: `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, `maintMarginBps 10`, `initMarginBps 20`, `fadMarginBps 300`, `baseCarryBps 500`, `minBountyAssets 1e6`, `bountyBps 10`, `keeperShareBps 5000`, `protocolShareBps 0`.

The headline test, `test_H02b_flatKillTakesTheLossFromTheDepositor`, opens a 100,000 USDG SHORT against a 200,000 USDG PEER long. The basket then moves 2%, from 1.00 to 0.98. The test calls this the `adverse` mark, and the logs show it moves against SHORT.

| Account / state | At commit | Twin: order fills at its first tick, closed at the adverse mark | Kill: trader withdraws free cash, self-executes, order fails | Kill minus twin |
|---|---|---|---|---|
| Junior LP value (`depositor twin` / `depositor after kill`) | not logged | 997,399.591652 USDG | 997,199.591651 USDG | **-200.000001 USDG** |
| Trader, clearinghouse balance + USDG token balance (`trader twin` / `trader after kill`) | 50,000.000000 USDG (asserted) | 49,720.799999 USDG | 50,000.000000 USDG | +279.200001 USDG (loss avoided) |
| Order status | pending | Executed | Failed, `kill terminal reason 7` (PlannerRejected) | |

**Worst harm**, `test_F1_worstHarm_largestOrderBothTranches`: a 390,000 USDG SHORT against a 390,000 USDG PEER long, with a 5% move.
- Junior: `owed junior 983038972889` against `held junior 982258972888`, a gap of **-780.000001 USDG**.
- Senior: unchanged (`owed senior 1000009146` = `held senior 1000009146`).
- Trader: `trader loss avoided 1084200001` (1,084.200001 USDG).

**Both branches**, `test_F1_combo_o11F04_decideAtSecond31_bothBranches`: the same 100,000 USDG order, decided 31 seconds after the blocker's commit.
- Kill branch: `depositor after kill 997199591678` against `depositor in the twin 997399591677` (-199.999999 USDG).
- Keep branch, at a 2% favourable move: `trader whole, start 50000000000` becomes `trader whole, favourable 51913130026` (+1,913.130026 USDG, net of fees).

**Economics, so the reviewer can judge severity.** All figures come from the release log. The arithmetic is mine.
- Per 100,000 USDG order at a 2% move:
  - Adverse branch: the Junior LP loses 200.000001 USDG against the twin, and the trader avoids a 279.200001 USDG loss.
  - Favourable branch: the trader keeps +1,913.130026 USDG.
  - The trader has no losing branch.
- Worst harm shown: 780.000001 USDG to the Junior LP on the 390,000 USDG order at 5%. The trader avoids 1,084.200001 USDG.
- Capital:
  - The order margin is 300.000001 USDG (`minimal margin 300000001`).
  - The trader account holds 50,000 USDG (450,000 USDG in the worst-harm test).
  - The free cash that the kill removes must exceed the order's VPI reserve. The test does not log that amount separately.
- Avoided loss against capital, per adverse cycle: 279.200001 / 300.000001 = 93% of the order margin, or 0.56% of the 50,000 USDG account.

**Preconditions and bounds.**
- The open must reduce skew, so an opposite book must exist. It must also carry the minimal admitted margin, which the test finds by binary search on `previewOpenRevertCode`.
- No other executor may fill the order at its first tick before the trader withdraws. Plether operates the keepers. A keeper that executes first fills the order normally, and the option is lost for that cycle. The combo test removes this race by holding the FIFO head for 30 seconds with an order whose first tick fails validation. That composes with a separate issue and is not needed for the kill itself.
- Each cycle needs a new commit. The tests each show one cycle.
- The keep branch alone overlaps known-issues id 1567. The kill branch is the part this report claims.
- Containment: `EMERGENCY_RESPONSE_GUIDE.md:14` says: "The guardian can add restrictions but cannot remove them, change configuration, set prices, move funds, or invoke arbitrary targets." The guardian's Router risk-off (`EMERGENCY_RESPONSE_GUIDE.md:28`) may stop new commits. That was not tested. No pause or owner action returns value already lost.
- Deployment: `DEPLOYMENT.md:20` says: "Deploy the Engine, Clearinghouse, Router, Books, Oracle, Pool, vaults, and sidecars from one reviewed source revision." The PoC deploys every in-scope contract from `a82d4fef` through the repository's own `BasePerpTest` harness.

## Recommendation

At commit, reserve the free settlement that the admission preview relied on until the order executes or terminates. At minimum, reserve the VPI rebate reserve that `CfdEnginePlanLib.sol:564-567` draws from free cash. Alternatively, make `validateWithdraw` refuse a withdrawal that would make a pending open of the same account fail.

## References

- Blamed function: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L404-L454
- VPI reserve from free settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L561-L571
- Flat-account withdraw guard: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L260-L272
- Execution-time free settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol#L153-L162
- Self-execution bounty refund: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L336-L337
- Design statements: `packages/perps/SECURITY.md:647`, `:658-659`
- PoC: `F74Release.t.sol` and its log `forge-F74Release.log`, both in the Gist. The PoC is a local Foundry run through the repository's `BasePerpTest` harness at the release risk parameters. Only Pyth (`MockPyth`) and USDG are mocked. No fork block is used.


## Link to Proof of Concept

https://gist.github.com/tdev00/817e0fe312b2b22b9783db62e4c0c95c

## Proof of Concept

## Proof of Concept

**Attack, step by step, as the tests execute it on the `BasePerpTest` harness at the release risk parameters.**
1. The harness seeds the pool. The test contract is the Junior LP, with a 1,000,000 USDG Junior deposit plus a 1,000 USDG seed (`BasePerpTest.sol:186-189`, `:247-249`, `:255-257`). The harness also seeds 1,000 USDG of Senior. `TRADER` (0xA11CE) and `PEER` (0xB0B) are each funded with 50,000 USDG.
2. `PEER` deposits 200,000 USDG more and opens a 200,000 USDG LONG, which a keeper fills at 1.00. Two seconds pass and the mark refreshes at 1.00.
3. `TRADER` finds the smallest margin the commit preview admits for a 100,000 USDG SHORT (a skew-reducing open). It binary-searches `engineLens.previewOpenRevertCode` and gets `minimal margin 300000001` (300.000001 USDG). `TRADER` then commits the open with `router.commitOrder` (execution window 60 s). The commit passes `_validatedOpenBounty` (`router/OrderLiquidationBatchLogic.sol:431-445`) because the account's free cash covers the VPI reserve. The test records the trader's clearinghouse balance plus its USDG token balance at commit: 50,000 USDG.
4. The test calls `vm.snapshotState()`. Two branches then run from the same state:
   - **Twin (what the order owes).** `TRADER` executes the order at its first tick (1.00), and it fills (`Executed`). `TRADER` commits a close, and a keeper executes it at 0.98, the `adverse` mark, which moves against SHORT. Junior LP value: `depositor twin 997399591652`. Trader total: `trader twin 49720799999`.
   - **Kill (the attack).** The test calls `vm.revertToState(snap)`. `TRADER` reads `getAccountAssetBuckets(TRADER).freeSettlementAssets` and calls `clearinghouse.withdraw(TRADER, removed)`. The withdrawal is allowed because the account is flat (`CfdEngineSettlementSidecar.sol:260-272`). `TRADER` then executes its own order at the same first tick (1.00). The order fails (`Failed`, `kill terminal reason 7` = `PlannerRejected`), and the bounty returns to `TRADER` (`OrderRouter.sol:336-337`). The mark then moves to 0.98. Junior LP value: `depositor after kill 997199591651`. Trader total: `trader after kill 50000000000`.
5. The test asserts two things:
   - The trader is exactly whole after the kill (`assertEq(traderKill, internalStart + walletStart)`).
   - The Junior LP ends below the twin (`assertLt(juniorLP, owed)`).

   Kill minus twin: Junior LP -200.000001 USDG, trader +279.200001 USDG.
6. Two more tests run the same attack:
   - `test_F1_worstHarm_largestOrderBothTranches` repeats steps 2-5 with a 390,000 USDG order at a 5% move and reads both tranches.
   - `test_F1_combo_o11F04_decideAtSecond31_bothBranches` makes the decision 31 seconds after the blocker's commit. It also runs the favourable branch (keep): the order fills at the old tick and closes at 1.02.

The tests use these cheatcodes only:
- `vm.prank` for each party's own account
- `vm.warp` and `vm.roll`
- `vm.snapshotState` and `vm.revertToState`, to compare branches from one state

Only Pyth (`MockPyth`) and USDG (the harness settlement token) are mocked, which the rules permit.

**Risk parameters of the test contracts.** `A8Risk`, `A8Risk_a8_A8Commit3` and `A8Risk_a8_A8Commit4` use identical parameters, and `_vpi()` returns `0.01e18` in all three test contracts. These are the release values in `DEPLOYMENT.md:131-141`:

```solidity
function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
    return CfdTypes.RiskParams({
        vpiFactor: _vpi(),
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
```

**Runnable tests.** The Gist holds these files:
- `F74Release.t.sol`: three Foundry tests (contracts `A8VpiTest`, `A8ChaseTest`, `A8LastTest`), run through the repository's `BasePerpTest` harness.
- `forge-F74Release.log`: the captured forge log.
- `TriageRouterB.t.sol` and `TriageRouterB.log`: the original run of the same tests at the harness-default margins.

All figures in this report come from `forge-F74Release.log`. Gist: the secret Gist linked in the PoC Link field

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules included). Install solc 0.8.35, the repository's pinned compiler.
2. Create `poc/` at the repository root and copy `F74Release.t.sol` into it. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`. The file imports the harness as `../packages/perps/test/perps/BasePerpTest.sol`.
3. Run from the repository root, with the test directory pointed at `poc/`:
   ```
   export FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]'
   forge test --match-path poc/F74Release.t.sol --match-test 'test_H02b_flatKillTakesTheLossFromTheDepositor|test_F1_worstHarm_largestOrderBothTranches|test_F1_combo_o11F04_decideAtSecond31_bothBranches' -vv
   ```
   Expected: `3 tests passed, 0 failed, 0 skipped (3 total tests)`. No RPC or fork block is needed. The harness deploys every in-scope contract locally from the pinned source.

Headline test, `contract A8VpiTest` (`F74Release.t.sol:1121-1194`), with its margin helper:

```solidity
/// @dev Smallest margin the commit preview admits while the account still holds free cash.
function _minimalMargin(
    address who,
    CfdTypes.Side side,
    uint256 size
) internal view returns (uint256 lo) {
    uint256 hi = 30_000e6;
    lo = 0;
    uint256 price = engine.lastMarkPrice();
    while (lo + 1 < hi) {
        uint256 mid = (lo + hi) / 2;
        if (engineLens.previewOpenRevertCode(who, side, size, mid, price) == 0) {
            hi = mid;
        } else {
            lo = mid;
        }
    }
    lo = hi;
}

/// @dev H02 flat: a skew-reducing open whose margin sits exactly at the bar while free cash tops the VPI reserve.
///      The owner removes free cash after the tick is known and self-executes the failure.
///      Owed to the junior depositor: the same as a twin where the order fills at its first tick and takes the loss.
function test_H02b_flatKillTakesTheLossFromTheDepositor() public {
    _deposit(PEER, 200_000e6);
    _openPos(PEER, CfdTypes.Side.LONG, 200_000e18, 20_000e6);
    vm.warp(vm.getBlockTimestamp() + 2);
    _refresh(ONE);
    uint256 size = 100_000e18;
    uint256 margin = _minimalMargin(TRADER, CfdTypes.Side.SHORT, size);
    console2.log("minimal margin", margin);
    OrderV3Types.OrderRequest memory r = _openReq(CfdTypes.Side.SHORT, size, margin);
    uint64 id = _commit(TRADER, r);
    uint256 walletStart = settlementToken.balanceOf(TRADER);
    uint256 internalStart = clearinghouse.balanceAssets(TRADER);
    uint256 adverse = ONE - ONE / 50;

    uint256 snap = vm.snapshotState();

    // Twin: the order fills at its first tick, then the Trader closes at the adverse mark.
    OrderV3Types.ExecutionResult memory fill = _execBy(TRADER, id, ONE);
    assertEq(uint8(fill.status), uint8(OrderV3Types.LifecycleStatus.Executed), "twin fills at the margin bar");
    OrderV3Types.OrderRequest memory c = _closeReq(CfdTypes.Side.SHORT, size);
    uint64 closeId = _commit(TRADER, c);
    _exec(closeId, adverse);
    uint256 owed = _depositor();
    uint256 twinTrader = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
    console2.log("depositor twin", owed);
    console2.log("trader twin", twinTrader);

    vm.revertToState(snap);

    // Kill: the Trader saw the adverse move and removes the free cash the fill needed.
    uint256 removed = clearinghouse.getAccountAssetBuckets(TRADER).freeSettlementAssets;
    vm.prank(TRADER);
    clearinghouse.withdraw(TRADER, removed);
    OrderV3Types.ExecutionResult memory killed = _execBy(TRADER, id, ONE);
    console2.log("kill terminal reason", uint256(killed.terminalReason));
    assertEq(uint8(killed.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the owner's withdrawal kills it");
    assertEq(_size(TRADER), 0, "no exposure");
    _refresh(adverse);
    uint256 juniorLP = _depositor();
    uint256 traderKill = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
    console2.log("depositor after kill", juniorLP);
    console2.log("trader after kill", traderKill);
    assertEq(traderKill, internalStart + walletStart, "the kill cost the Trader nothing");
    assertLt(juniorLP, owed, "depositor misses the loss the order would have paid");
}
```

The helpers it calls are in the same file (`F74Release.t.sol:918-1119`): `_openReq`, `_closeReq`, `_commit`, `_execBy`, `_exec`, `_openPos`, `_refresh`, `_depositor`, `_deposit` and `_size`. `_execBy` passes `_mockPythUpdateData(price)` to `router.executeOrder` under the named caller's prank. `_depositor()` returns `juniorVault.convertToAssets(juniorVault.balanceOf(address(this)))`.

Captured output, from `forge-F74Release.log` in the Gist. Values are in USDG with 6 decimals. The log shows the run's folder as `poc-f74/`; the steps above use `poc/`.

```
Ran 1 test for poc-f74/F74Release.t.sol:A8ChaseTest
[PASS] test_F1_worstHarm_largestOrderBothTranches() (gas: 22713447)
  owed junior 983038972889
  held junior 982258972888
  owed senior 1000009146
  held senior 1000009146
  trader loss avoided 1084200001

Ran 1 test for poc-f74/F74Release.t.sol:A8LastTest
[PASS] test_F1_combo_o11F04_decideAtSecond31_bothBranches() (gas: 29035920)
  depositor after kill 997199591678
  depositor in the twin 997399591677
  trader whole, start 50000000000
  trader whole, favourable 51913130026

Ran 1 test for poc-f74/F74Release.t.sol:A8VpiTest
[PASS] test_H02b_flatKillTakesTheLossFromTheDepositor() (gas: 21831594)
  minimal margin 300000001
  depositor twin 997399591652
  trader twin 49720799999
  kill terminal reason 7
  depositor after kill 997199591651
  trader after kill 50000000000

Ran 3 test suites in 91.16ms (229.61ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

Reading the logs:
- **Headline:** the Junior LP ends 200.000001 USDG below the twin (997,399.591652 vs 997,199.591651). The trader stays whole at 50,000.000000 instead of 49,720.799999, avoiding a 279.200001 USDG loss.
- **Worst harm:** the Junior LP ends 780.000001 USDG below the twin. Senior is unchanged at 1,000.009146. The trader avoids 1,084.200001.
- **Both branches:**
  - Kill: the Junior LP ends 199.999999 USDG below the twin.
  - Keep: the trader goes from 50,000.000000 to 51,913.130026 (+1,913.130026).

