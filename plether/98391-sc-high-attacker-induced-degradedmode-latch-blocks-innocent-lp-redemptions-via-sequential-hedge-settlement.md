# \#98391 [SC-High] Attacker-Induced DegradedMode Latch Blocks Innocent LP Redemptions via Sequential Hedge Settlement

**Submitted on Nov 7th 2026 at 15:50:55 UTC by @mscter834kd for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98391
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`CfdEngine` reserves pool liability as `L = max(longSide.maxProfit, shortSide.maxProfit)`, using each side's maxProfit pinned at entry. An unprivileged trader can open a delta-neutral LONG/SHORT hedge sized near pool capacity and, after a market move, close the winning leg first while the losing leg is still open. The winning-leg payout transiently drops effective assets below that pinned liability (`E < L`), which latches `degradedMode`. The attacker then closes the losing leg; afterwards the pool holds zero open positions and zero remaining max-profit liability, yet the latch stays active (it is monotone and only the owner can clear it). While latched, innocent LP redemption/deposit funding and new risk-increasing trader orders are blocked pool-wide. The attack does not require the trader to profit, but it prevents unrelated LPs from funding matured redemptions and blocks new risk-increasing trading until the owner intervenes.

## Vulnerability Details
Maximum directional liability is pinned at entry and does not track the live mark:
- `L = max(globalLongMaxProfit, globalShortMaxProfit)` in `CfdEngine._maxLiability()` (`packages/perps/src/CfdEngine.sol:1044`); side maxProfit is fixed at open (`CfdMath.calculateExactMaxProfit`).
- Admission requires post-op `E >= L + B`, where `E = max(pool.totalAssets() - totalTraderClaimBalance, 0)` and `B = ceil(L * settlementBufferBps / 10_000)`, `settlementBufferBps = 25` (`SECURITY.md:339`). A balanced hedge has net skew ~0, so it bypasses the one-sided skew cap and lets `L` be sized up to ~`E - B`.

After a close, the engine pays the realized credit from the pool and then checks solvency:
- `CfdEngine._applyClose()` (`CfdEngine.sol:1306-1321`) lowers `E`, then calls `_enterDegradedModeIfInsolvent()` (`CfdEngine.sol:1351-1369`): if `E < L` it sets `degradedMode = true`.
- Closing the **winning** leg while the opposite leg is still open leaves that side's **full pinned maxProfit** in `L`. So `E` can fall below `L` even though the surviving leg is only ~3% underwater and the pool can still honor every current-price obligation. That is a worst-case liability figure, not an actual insolvency.

The latch is monotone and recovery is privileged-only:
- `clearDegradedMode()` (`CfdEngine.sol:658-666`) is `onlyOwner` and requires `E >= L`. There is no automatic clear and no permissionless clear. If the attacker keeps the losing leg open, the owner must first `recapitalizePool(...)` (`CfdEngine.sol:632`) to restore `E >= L`, then clear.

While latched, `degradedMode` blocks unrelated users:
- LP redemption/deposit funding: `HousePool._requireWithdrawalsLive()` (`HousePool.sol:1864-1869`, called in `_settleLpEpoch`, reverts `HousePool__DegradedMode`) and `_entryStatusBlocked()` (`HousePool.sol:1143`).
- New risk-increasing trader opens: `CfdEngineOpenQuoter.sol:112` (surfaced at commit as `OrderRouter__DegradedMode`).
- Trader withdrawals while a position is open: `CfdEngineSettlementSidecar.sol:274-275`.
Protective transitions (closes, liquidations) deliberately stay live.

Root cause: a transient, worst-case under-reservation that the attacker manufactures purely by close ordering, on a pool that is solvent at current prices, latches a persistent protocol-wide availability block against unrelated LPs and traders, clearable only by a privileged owner action.

## Impact Details
Griefing: the attack inflicts an availability block on unrelated users and does not depend on the attacker profiting.
- A genuine matured innocent LP redemption that would pay **1,004,948,912 (1,004.95 USDG)** cannot be funded while latched (`settleLpEpoch` reverts `HousePool__DegradedMode`, selector `0x9f7adff3`). All pending LP redemptions/deposits are blocked together; new risk-increasing trader opens and withdrawals-with-open-position revert (`OrderRouter__DegradedMode`, `0x22a6276e`). The blocked amount scales with victim redemption size and total pending LP demand, bounded by pool depth. (Figures from the end-to-end six-feed test.)
- The attack does not depend on extracting funds from the protocol. The attacker opens opposing positions and incurs execution-related costs to trigger an availability restriction affecting unrelated users. The E2E PoC demonstrates the disruption, but does not assert an exact net attacker profit or loss.
- No user funds are demonstrated stolen. The security impact is the loss of availability (unrelated LPs cannot get their matured redemptions funded, and new risk-increasing trading is blocked), not the transient liability shortfall itself.

Recovery and limitations:
- The latch is owner-recoverable. After both legs close, the pool is solvent and the owner clears it in one transaction (shown in the PoC). Even with the losing leg held open, the owner can `recapitalizePool(...)` then `clearDegradedMode()`; the recapitalization needed is on the order of the liability shortfall at the moment of latch (`L - E = ~57.48 USDG` in the end-to-end test). A sustained interruption therefore depends on the owner's response and ability to restore the required liability coverage; this report does not claim a guaranteed fixed-duration freeze.
- The trigger needs a real market move. The end-to-end test uses a modeled 3% six-feed basket move; the latch fires once the winning-leg payout exceeds the pool's buffer headroom. The attacker cannot force the move, and cannot cause fund loss.

## References
- `packages/perps/src/CfdEngine.sol`: `_maxLiability` L1044, `_applyClose` L1306-1321, `_enterDegradedModeIfInsolvent` L1351-1369, `clearDegradedMode` L658-666, `recapitalizePool` L632
- `packages/perps/src/HousePool.sol`: `_requireWithdrawalsLive` L1864-1869, `_entryStatusBlocked` L1143
- `packages/perps/src/CfdEngineOpenQuoter.sol:112`, `packages/perps/src/CfdEngineSettlementSidecar.sol:274-275`
- `packages/perps/SECURITY.md`: buffered entry solvency L339, degraded containment L340
- Frozen commit: a82d4fefb02768e5b3bddcc97dee371eee24fea8


## Link to Proof of Concept

https://gist.github.com/Rhasta0323/4a8a8b62287f8b6dc4a4a20eb6d7ab2c

## Proof of Concept

## Proof of Concept

A complete, runnable Foundry PoC is provided in the secret Gist linked in this submission. It is test-only; no production `src` is modified. The only non-test change is adding the `virtual` keyword to four oracle-feed-config helpers in the test harness `packages/perps/test/perps/BasePerpTest.sol` (no behavior change, since all existing tests use the default 2-feed config), so the end-to-end test can supply the production six-feed basket. The Gist contains `poc.patch`, which adds the end-to-end test and that 4-keyword harness change, plus `REPRODUCE.md` and the captured run log. I verified this reproduces from a fresh checkout of the frozen commit (`git clone`, `git checkout a82d4fef`, `git apply poc.patch`, `forge test`), with `src` untouched.

### Reproduce
1. `git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether`
2. `git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8` (and `git submodule update --init --recursive` if the lib/ submodules are missing)
3. `git apply poc.patch`
4. `cd packages/perps && forge test --match-contract DegradedModeGriefE2E -vv` -> 2 passed, 0 failed

The headline test `DegradedModeGriefE2E.t.sol` wires the entire real production stack (PletherOracle + OrderRouter + execution sidecars + CfdEngine + MarginClearinghouse + HousePool + TrancheVault) using the exact six-currency launch basket (EUR/USD, USD/JPY^-1, GBP/USD, USD/CAD^-1, USD/SEK^-1, USD/CHF^-1 with the launch weights and base prices), MockPyth as the only external price provider, nonzero (~1 bp) feed confidence, launch adverse pricing (2000 bps), launch carry (500 bps), and the engine-default launch margins / execution fee (4 bps) / settlement buffer (25 bps).

### Starting state (identical for baseline and attack)
- Pool: junior 5,000 USDG (LP_J 4,000 + innocent LP_J2 1,000) + senior 1,500 USDG.
- Innocent LP_J2 submits a legitimate full redemption (`juniorVault.requestRedeem`), matured across normal epochs.
- Attacker funds two accounts and opens a delta-neutral hedge: 64 lots LONG + 64 lots SHORT in alternating 16-lot chunks (net skew ~0, so the one-sided skew cap is bypassed; admitted since `L ~= 6.40e9 <= E - B`).
- The six-feed basket anchors exactly at 1e8 (neutral mark = 100,000,000).

### Attack branch (test_e2e_attack_six_feed)
1. Apply an internally consistent uniform-USD six-feed -3% move (every currency scaled by q/1e8). Engine mark goes 100,000,000 -> 97,000,000. LONG wins, SHORT loses.
2. Before the winning close: E = 6,502,001,760; pinned liability L = 6,399,872,256.
3. Close the WINNING leg (LONG) first, through the public Router, while the SHORT is still open. The pool pays the LONG profit and E drops to 6,342,388,088. The surviving SHORT's pinned maxProfit keeps L = 6,399,872,256, so now E (6,342,388,088) < L (6,399,872,256) and `degradedMode` latches. Assert `engine.degradedMode() == true`.
4. Close the LOSING leg (SHORT). Now there are zero open positions and `maxLiability == 0`, yet `degradedMode` is still true. Assert positions == 0, maxLiab == 0, `degradedMode == true`.
5. The keeper tries to fund the innocent LP_J2 redemption: `router.settleLpEpoch(...)` reverts with exactly `HousePool__DegradedMode` (selector `0x9f7adff3`), and `juniorVault.claimableRedeemAssets(reqId, LP_J2) == 0`. The matured redemption is inaccessible (asserted).
6. Recovery: the pool is solvent, so the owner calls `clearDegradedMode()`, which succeeds. Assert `degradedMode == false`.

### Baseline branch (test_e2e_baseline_six_feed): identical start and move, only the close ORDER differs
1. Same hedge, same -3% move.
2. Close the LOSING leg (SHORT) first, then the winning leg (LONG). E goes 6,502,001,760 -> 6,526,776,937 and `degradedMode` never latches. Assert `degradedMode == false`.
3. The keeper funds the innocent LP_J2 redemption: `settleLpEpoch` succeeds, and LP_J2 claims and collects **1,004,948,912 (1,004.95 USDG)** (asserted).

The only difference between the two branches is attacker-controlled close ordering, which isolates the order-dependent latch and the resulting innocent-LP availability block. The test runs at launch parameters (carry 500 bps, settlement buffer 25 bps, execution fee 4 bps, adverse confidence 2000 bps) and against the exact six-feed launch basket.

Both tests pass: 2 passed, 0 failed.
