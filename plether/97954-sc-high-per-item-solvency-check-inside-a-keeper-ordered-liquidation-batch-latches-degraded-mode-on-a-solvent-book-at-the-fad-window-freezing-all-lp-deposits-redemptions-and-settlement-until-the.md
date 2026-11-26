# \#97954 [SC-High] Per-item solvency check inside a keeper-ordered liquidation batch latches degraded mode on a solvent book at the FAD window, freezing all LP deposits, redemptions and settlement until the owner intervenes

**Submitted on Nov 4th 2026 at 03:29:08 UTC by @risinghorse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97954
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief / Intro

Plether enters "degraded mode" — which blocks LP epoch settlement, LP entry, and LP/trader exits — whenever a close or liquidation leaves effective equity `E` below the larger side's maximum-profit liability `L`. That check runs **after each individual item** of a permissionless liquidation batch, and the batch executes accounts in the **caller's chosen order** with no sorting. In the 30-minute frozen-adjacent (FAD) window before the Friday close the liquidation threshold is 3%, so ordinary ~1% accounts become liquidatable at essentially no cost. A keeper (anyone) can therefore submit a batch that liquidates a *winning* account first: paying the winner drops `E` while `L` is still set by the opposing *losing* side, so the per-item check sees a false `E < L` and latches — even though the pool is solvent, LP NAV is intact, and the offsetting loss becomes cash in the same transaction. The latch has no permissionless exit (`clearDegradedMode` is owner-only), so every LP deposit, redemption and settlement is frozen until the owner acts. The PoC shows the book ending **empty and over-collateralized** (`E` far above `L=0`) yet still frozen more than 24 and 73 hours later, with the pool's entire cash sitting idle.

## Vulnerability Details

**The check runs on a transient intermediate state.** `_enterDegradedModeIfInsolvent` ([`CfdEngine.sol:1351-1368`](../../../src/CfdEngine.sol)) compares current `E = pool.totalAssets() − totalTraderClaimBalanceAssets` (`_availableCashForFreshPoolPayouts`, `:865-872`) against `L = max(longMaxProfit, shortMaxProfit)` (`_maxLiability`, `:1044-1050`):

```solidity
uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
...
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

It is called after **every** close (`:1320`) and liquidation (`:1344`). When a *winning* account is paid, `E` falls by the payout, but `L` is still the *opposing* (losing) side's max profit, which has not yet been liquidated. So the check fires with no real insolvency: the pool's NAV is unchanged (the opposing position's loss is a marked receivable that becomes realized cash in the very same batch), yet `degradedMode` latches.

**The keeper picks the order.** `executeLiquidationBatch` ([`router/OrderLiquidationBatchLogic.sol:738-790`](../../../src/router/OrderLiquidationBatchLogic.sol)) walks `accounts[nextIndex]` in the caller's order with no sorting. Liquidating the winner before the loser latches; losers-first or alternating order does not. Liquidation is permissionless and "harm caused by any caller acting as a keeper is in scope."

**The FAD window makes the trigger cheap.** From 30 minutes before the Friday close the liquidation threshold is the 3% FAD margin, so an ordinary account posted at ~1% margin is liquidatable. Liquidation charges no VPI and no execution fee; only the ~5 bps LP share offsets the payout, so the latch trips at a price move of only ~0.32% (measured). Guardian containment does **not** pre-empt it: liquidations are unpausable, so a full-containment guardian action still allows the latching batch (`test_S4`).

**The latch has no permissionless exit.** `degradedMode = false` is written only by `clearDegradedMode` ([`CfdEngine.sol:658-665`](../../../src/CfdEngine.sol)), which is `onlyOwner`. While latched, `settleLpEpoch` reverts via `_requireWithdrawalsLive` ([`HousePool.sol:832`, `:1864-1868`](../../../src/HousePool.sol)) and LP entry is blocked (`:1109/:1128/:1143`). Matured redemption and committed-Senior escrow cannot be cancelled (`TrancheVault.sol:857`, `:896`), and a trader with a live position cannot withdraw while latched. So every LP exit, entry and settlement is frozen until the owner clears it.

**The sequence the PoC performs (Variant A — no attacker capital):**
1. An organic two-sided book sits near capacity (`E ≈ L + B`) with both sides under the 3% FAD margin — reached by ordinary trading plus any LP redemption backlog.
2. At the FAD window, any caller submits `executeLiquidationBatch([winner, loser], pythUpdate)`.
3. The winner is paid first → `E` drops below `L` → `degradedMode` latches.
4. The loser is then liquidated in the same batch → `L → 0`, `E` far above `L` — but the latch stays set.
5. With the opposite order (losers first) there is no latch. The attacker only needs to be the first keeper and to order winners first.

Variant B uses the attacker's own delta-neutral pair (long at ~1% margin, short at ~4%) so it does not depend on an organic book; a ≥0.31% move then latches on the winner's liquidation.

## Impact Details

All figures from `LatchStrictVerify.t.sol` on the strict release fixture (production `OrderRouter`, six release feeds, release parameters asserted on-chain), USDG. `E` = effective equity, `L` = larger side's max profit, `B` = 25 bps settlement buffer.

**Variant A — organic book, any keeper (`test_S1`):**
- After an LP drain to `E = 1,002,503`, `L = 1,000,003` (i.e. `E ≈ L + B`).
- Control, **losers-first**: no latch; a 99,518 redemption funds Friday night. **Alternating** order: no latch.
- Attack, **winners-first**: the latch fires on an intermediate state `E = 999,532 < L = 999,982` at a 0.40% move; the keeper earns the ordinary bounty of 995. The batch then empties the book: `E = 1,004,298`, `L = 0`, firewall free cash 1,004,298.
- **24 hours and 73 hours later, with no owner action:** every LP exit still reverts, both max-withdraw views are 0, 400,000 of deposit escrow has no refund path, and the pool's full 1,004,298 of cash is idle. Because `L = 0` leaves all cash free under the firewall, the block is the latch itself, **not** a liquidity shortage.
- Still working while latched: an already-funded redemption (997,498) remains claimable; an unmatured Junior deposit can still cancel (10,000). A guardian settlement hold refunds Senior escrow (100,000) but Junior escrow stays stuck. After the owner clears, the queued redemption funds (100,225).
- Minimum move (`test_S1b`): no latch at 0.26–0.30%, latch at **0.32%**.

**Variant B — own hedged pair at Senior-cap scale (`test_S2`):** pool `E = 49,900,008`, `L = 49,775,434`; attacker margin 2,349,407 (returned). Liquidating the winner alone latches at a 0.31% move. If the attacker keeps the loser open, `clearDegradedMode` reverts `StillInsolvent` and the owner must inject ~22,161 of recap; total attacker cost incl. exit 102,117 (20.51 bps of `L`), against 49,942,458 of pool value frozen — a 489:1 ratio of frozen funds to attacker cost.

**Severity (honest):**
- **High** on a literal reading: "temporary freezing of funds for at least 24 hours," measured at 24h and 73h with no owner action; the program assumes no response/detection deadline unless documented, keepers are untrusted, and the firewall shows all cash free so the "lack of liquidity" exclusion does not apply.
- **Medium** is the realistic fallback: in Variant A the owner clears the latch in a single, capital-free `clearDegradedMode` call (`E ≥ L` holds once the book is empty), no funds are lost, and the attacker gains only the ordinary keeper bounty (a griefing shape). A reviewer who credits a prompt owner response lands here.
- Variant B lets the attacker *control the duration* by keeping the loser open (the owner must recapitalize to clear), at a 489:1 frozen-to-cost ratio, which strengthens the ≥24h claim but costs real capital.
- **Disclosed-design risk:** degraded mode, its blocking of LP settlement, and the owner-only clear are all documented (SECURITY.md "Solvency and containment"; ACCOUNTING_SPEC.md degraded-mode section; WHITEPAPER.md). What is **not** disclosed is that a *caller-chosen order* produces a *false* latch on a solvent book with NAV intact, at near-zero cost, through a transient intermediate state. Frame the root cause as "a per-item check on a caller-ordered transient state latches without insolvency," not as "degraded mode exists."

No funds are stolen and solvency is never broken; this is purely a temporary freeze of LP funds (and of trader exits for accounts with live positions).

## References

- `packages/perps/src/CfdEngine.sol:1351-1368` — `_enterDegradedModeIfInsolvent`; `:1320` and `:1344` call it after each close/liquidation; `:865-872` (`E`), `:1044-1050` (`L`), `:658-665` (owner-only `clearDegradedMode`, the only `degradedMode = false` writer).
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:738-790` — `executeLiquidationBatch` runs accounts in caller order with no sorting.
- `packages/perps/src/HousePool.sol:832`, `:1864-1868` — `settleLpEpoch` reverts `HousePool__DegradedMode` while latched; `:1109/:1128/:1143` block LP entry.
- `packages/perps/src/TrancheVault.sol:857` (`SeniorDepositCommitted`), `:896` (`RedeemCancellationUnavailable`) — matured escrow cannot be cancelled.
- Documented context (for the design discussion): SECURITY.md "Solvency and containment"; ACCOUNTING_SPEC.md degraded-mode section; EMERGENCY_RESPONSE_GUIDE.md (liquidations unpausable).
- Known-issue check: not in the Cecuro AI audit, SC pre-audit or AI scan, nor in the named SECURITY.md known-limitation sections. The first hunt's internal Medium (a hedged profitable *close* latching, see `DegradedLatchProdConfig.t.sol`) is the same latch via a different trigger; this report's keeper-order batch trigger on a solvent book and the empty-book persistence are the new elements.

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the in-scope commit, no in-scope contract modified; only Pyth and USDG are mocked. All steps use public entrypoints and EOA signers; the only owner call is the documented `clearDegradedMode`. No `vm.store`, `vm.etch`, `mockCall`, or prank as router/engine/pool/vault.

**File:** `packages/perps/test/audit_poc/fad_latch/LatchStrictVerify.t.sol` (with the shared release fixture it imports). 8/8 tests pass. The hedged-profitable-close variant of the same latch is in `packages/perps/test/audit_poc/DegradedLatchProdConfig.t.sol`.

**Run:**
```bash
cd packages/perps && "$HOME/.foundry/bin/forge" test --match-path "test/audit_poc/fad_latch/LatchStrictVerify.t.sol" --skip "test/perps/**/*.t.sol" --skip "script/**" -vv
```

**Key tests and what they show:**
- `test_S1_StrictOrganicBook_KeeperOrder_FreezeAndControls` — Variant A: winners-first latches at `E = 999,532 < L = 999,982`; losers-first and alternating do not; book ends empty (`L = 0`) yet LP exits are still blocked at 24h and 73h; 400,000 escrow stuck; funded claim still paid; owner clear then funds the redemption.
- `test_S1b_StrictThresholdSweep` — the minimum latching move is ~0.32%.
- `test_S2_StrictOwnHedgedPair_SeniorCap` — Variant B at 49.9M: 489:1 frozen-to-cost; keeping the loser open forces an owner recap to clear.
- `test_S4_GuardianContainmentDoesNotPreempt` — full guardian containment does not stop the latch (liquidations unpausable).
- `test_S5_WinnersOnly_TraderWithdrawGateAndOwnerRecoveryNeeds` — a trader with a live position cannot withdraw while latched; it can after closing.

### Suggested remediation (directional)

Evaluate the solvency/latch condition on the **final** post-batch state rather than after each item, or net the opposing side's realized change before testing `E < L`, so that a batch which ends solvent does not latch on a transient intermediate. Alternatively, allow `clearDegradedMode` (or an automatic clear) to run permissionlessly whenever `E ≥ L` holds, so a false latch on an ultimately-solvent book cannot freeze LP funds until the owner acts.
