# \#100048 [SC-High] Deliberate degraded-mode latch via a direction-neutral pair freezes ~2.0M USDG LP capital and all open-trader margin with owner-only recovery (`CfdEngine.sol`)

**Submitted on Nov 16th 2026 at 06:37:04 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100048
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

For about 3,200 USDG in execution fees, and with no price exposure while the hedge is open, any unprivileged trader can force Plether's `HousePool` into degraded mode while the book has no bad debt. Degraded mode then reverts every LP epoch settlement (~2.0M USDG of LP capital becomes non-redeemable) and blocks every open trader's margin withdrawal. The latch is not cleared automatically: it is still set after the attacker has fully exited and the book is flat and fully backed (`L = 0`, pool assets 2,014,895.20 USDG). The only recovery is the owner's `clearDegradedMode`, which reverts `CfdEngine__StillInsolvent` for as long as the attacker keeps the losing leg open. With that leg on its **ordinary** margin (~67.7k USDG, returnable; no reliance on the publicly disclosed zero-carry behaviour), the owner's clear still reverts on day 14. Topped up to full margin (~2.0M USDG, returnable), it still reverts on day 35. Short of an unrecoverable owner recapitalization, the freeze therefore lasts as long as the attacker chooses to hold, and it can be repeated after any clear.

## Vulnerability Details

Degraded mode is a solvency latch. After every close or liquidation, the engine latches `degradedMode = true` when post-operation effective assets `E` fall below max liability `L`. The latch is intended for a terminal insolvency transition. The defect is that the admission gate leaves only a 25 bps buffer above `L`, so a trader holding both sides of a hedge can choose exactly when `E` crosses below `L` with no bad debt, and the resulting latch has no permissionless or automatic clear once `E >= L` is restored.

```solidity
// packages/perps/src/CfdEngine.sol#L1358-L1367  (post-op latch)
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

```solidity
// packages/perps/src/CfdEngine.sol#L658-L667  (owner-only recovery, reverts while E < L)
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) revert CfdEngine__NotDegraded();
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) revert CfdEngine__StillInsolvent();
    degradedMode = false;
    emit DegradedModeCleared();
}
```

`L = max(longMaxProfit, shortMaxProfit)`, and admission only requires `E >= L + B`, with `B = settlementBufferBps = 25 bps` of `L` (`CfdEngine.sol:153`). When both sides of a pair are near `L`, any realized winner payout larger than `B` pushes `E` below `L` once the winner leg is closed. A pair holder controls exactly when that happens, and the payout is fully cash-settled, so no trader claim (`C`) is ever recorded — there is no actual insolvency, only the latch.

While latched, every LP epoch settlement reverts, and every open trader's withdrawal reverts:

```solidity
// packages/perps/src/HousePool.sol#L1865-L1869  (_requireWithdrawalsLive)
//   reverts HousePool__DegradedMode() -> settleLpEpoch cannot process any epoch
// packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276  (validateWithdraw)
//   reverts CfdEngine__DegradedMode() (0x7e9b3b04) for every account with an open position
```

### Affected locations
- `CfdEngine.sol` post-op latch — `packages/perps/src/CfdEngine.sol#L1358-L1367`
- `CfdEngine.clearDegradedMode` (owner-only, `StillInsolvent` gate) — `packages/perps/src/CfdEngine.sol#L658-L667`
- `CfdEngine.sol` admission buffer `settlementBufferBps = 25` — `packages/perps/src/CfdEngine.sol#L153`
- `HousePool._requireWithdrawalsLive` (LP settlement freeze) — `packages/perps/src/HousePool.sol#L1865-L1869`
- `CfdEngineSettlementSidecar.validateWithdraw` (open-trader margin freeze) — `packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276`

### Preconditions
- The pool holds normal LP capital (~2,002,000 USDG in the PoC, DEPLOYMENT.md release parameters loaded by `PocBase`).
- The attacker can open normal trading capacity up to the admission gate `E ≈ L + B`. This is ordinary capacity any trader can use; the margin is returned apart from fees.
- An ordinary basket move larger than `B` (25 bps of `L`, ≈1% gives −4,406.5 USDG in the PoC) occurs in either direction. The hedge removes direction risk until the winner leg is closed.
- No privileged role, leaked key, or social engineering is required. Keepers are untrusted under the program rules; the attacker is an ordinary trader (one account per side, or two accounts). No in-scope contract is modified.

### Attack path
1. Attacker opens equal LONG and SHORT legs at 1.00 in steps until the open gate `E >= L + B` refuses further size (1,997,000 lots per leg in the PoC; `(P − C) − L = +5,000 USDG = B`). The book is fully hedged — no net price exposure.
2. Attacker waits for an ordinary ~1% move in either direction and closes only the **winning** leg. The pool pays the winner in cash (`C = 0`), `E` drops below `L`, and `degradedMode` latches.
3. End state: `settleLpEpoch` reverts `HousePool__DegradedMode`; non-owner `clearDegradedMode` reverts; every open trader's `validateWithdraw` reverts `CfdEngine__DegradedMode` (`0x7e9b3b04`). ~2.0M USDG LP capital and all open-position margin are frozen.
4. The attacker keeps the **losing** leg open. While it is open, `E < L` holds and the owner's `clearDegradedMode` reverts `StillInsolvent`:
   - **Ordinary margin (CRK3d, no top-up):** the leg keeps its normal 67,680.86 USDG margin and pays carry. Carry flows into the pool and slowly closes the gap (4,406.50 → 811.29 USDG by day 14), but the owner's clear still reverts on days 1, 2, 3, 7 and 14, each time after carry is realized. It first succeeds at the day-21 sample. LP settlement reverts `HousePool__DegradedMode` on days 1 and 7. This variant does not depend on the zero-carry behaviour.
   - **Full margin (CRK3c):** the attacker tops the leg up to its max profit (1,997,989.38 USDG locked, returnable). It then pays zero carry (publicly disclosed, see Dedupe; used here only to lengthen the hold), and the owner's clear reverts on days 1, 7 and 35.
   The owner's only earlier exit is to recapitalize ≥ 4,395.88 USDG out of pocket (an `X − 1` recap still reverts). After that payment the attacker's leg is still open (size 1,997,000), so the cycle can be repeated.

Cost: ~3,179 USDG in execution fees (4 bps per execution) for the minimal run, plus returnable margin; ~3,200 USDG per repeat cycle after each owner clear. Holding the losing leg costs ~4.4k USDG of carry over the ordinary-margin hold, or ties up ~2.0M USDG of returnable capital in the full-margin hold (high capital is not a downgrade under the program rules). Once the winner leg is closed, the held leg carries ordinary price exposure, which the attacker manages like any other position. The structural fix (raising `settlementBufferBps`) is behind `CfdEngineAdmin`'s 72-hour timelock, so the owner cannot cheaply out-run repeats without halting all legitimate trading.

## Impact Details

This is **Temporary freezing of funds for at least 24 hours**: for ≈3,200 USDG in fees an unprivileged trader freezes all LP redemptions (~2.0M USDG of LP capital), every open trader's position-backed margin withdrawal, and all new opens, on a solvent book with no bad debt. The PoC shows the latch still set after 7 days with no owner action, and still set after the book is flat and fully backed. The attack is cheap to repeat until the 72h-timelocked buffer change lands.

**Attacker-controlled duration:** while the attacker keeps the losing leg open, `E < L` holds, no LP epoch settles, and no permissionless or automatic clear exists. The owner's only lever, `clearDegradedMode`, reverts `StillInsolvent`: on day 14 with the leg on ordinary margin (CRK3d), and on day 35 with it at full margin (CRK3c). The owner can regain control only by recapitalizing ≥ 4,395.88 USDG that it never recovers, and after paying it the attacker's leg is still open, so the cycle repeats. The freeze ends only when the attacker closes the losing leg or the owner recapitalizes; no user, keeper, or time-based path clears it.

**Funds at risk:** ~2,002,000 USDG of LP principal (pool capital from DEPLOYMENT.md release parameters in `PocBase`) plus the full position-backed margin of every open trader (proven in `CRK3b`: a healthy bystander holding a 1,000-lot LONG with 7,999.90 USDG of free buying power cannot withdraw 100 USDG while latched — `validateWithdraw` reverts `CfdEngine__DegradedMode`). All of it is frozen, not lost. The figure is the LP + open-position USDG in the pool at the attacked state; it is bounded by the pool's TVL at the time of attack. No external price source is needed (USDG is the 6dp settlement token, assumed $1 par per SECURITY.md). Competition reward is pool-distributed (Primary $56,000 / All-Stars $16,000 / Podium $8,000, USDC), not a percentage of funds at risk.

Who loses: every LP (redemptions frozen), every open trader (margin withdrawals frozen), and all new opens (blocked). The freeze of all LP redemptions and every open trader's margin withdrawal exceeds 24 hours (≥14 days measured on ordinary margin and ≥35 days at full margin, with no permissionless exit), meeting the "temporary freezing of funds for at least 24 hours" definition. LPs earn the attacker's fees and carry, so the funds are frozen, not stolen.

## Recommendation

1. Make recovery permissionless: allow anyone (or the post-op check itself) to clear degraded mode once `_availableCashForFreshPoolPayouts() >= _maxLiability()` and `C` is fully backed. The condition is objective and already computed in `clearDegradedMode`.
2. Do not latch when the shortfall comes only from a cash-settled payout on a hedged book — latch on recorded trader claims (`C` increase) or terminal write-offs, and use the non-latching `L <= E < L + B` buffer state for the hedge-induced `E < L`.
3. Alternatively, size the admission buffer so an ordinary move on a hedged book cannot take `E` below `L` (include the opposite side's max-profit, or raise `B` for books with both sides near `L`).

## References
- Source (in scope): `packages/perps/src/CfdEngine.sol`, `packages/perps/src/HousePool.sol`, `packages/perps/src/CfdEngineSettlementSidecar.sol` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Documented latch behavior (for dedupe): SECURITY.md:246, :340, :342, :942, :1506-1510; WHITEPAPER.md:1322-1342, :774
- Zero-carry amplifier for fully margined positions (publicly disclosed; not required for this finding — the ordinary-margin hold in CRK3d does not use it): plether-core PR #138

## Link to Proof of Concept

https://gist.github.com/Akashneelesh/ae6b1b8f098c5b9ae8efa72096097e27

## Proof of Concept

## Proof of Concept

The PoC runs in the repository's Foundry suite at the in-scope commit (program PoC rule). Tests inherit `PocBase` (DEPLOYMENT.md release parameters); only Pyth (`MockPyth`) and USDG are mocked; no in-scope contract is modified; no `vm.store`/`deal` state-fabrication cheats.

**Setup** (run from the repository root; `CRK3c`/`CRK3d` import `CRK3c`'s helpers, so keep all four files together in `packages/perps/test/perps/audit/`):
```bash
# Main latch + LP-freeze + flat-book persistence + owner-clear restore:
forge test --root packages/perps --match-path "test/perps/audit/CRK3_HedgedPairPartialClose.t.sol" -vv

# Bystander open-trader margin-withdrawal freeze:
forge test --root packages/perps --match-path "test/perps/audit/CRK3b_LatchBlocksTraderWithdraw.t.sol" --match-test CRK3b -vv

# Attacker-held freeze, full margin (owner clearDegradedMode reverts at d1/d7/d35):
forge test --root packages/perps --match-path "test/perps/audit/CRK3c_AttackerHeldFreeze.t.sol" -vv

# Attacker-held freeze, ORDINARY margin, no zero-carry dependence (owner clear reverts through d14):
forge test --root packages/perps --match-path "test/perps/audit/CRK3d_NormalMarginHeldFreeze.t.sol" --match-test CRK3d -vv
```

**Output:**
```
# CRK3_HedgedPairPartialClose.t.sol
[PASS] test_CRK3_Control_NeutralClose() (gas: 138398981)
[PASS] test_CRK3_Control_NoPair() (gas: 1532418)
[PASS] test_CRK3_HedgedPairWinnerLegClose() (gas: 139775304)
Suite result: ok. 3 passed; 0 failed; 0 skipped
#   ... asserts: after winner close (P−C)−L < 0, degraded=true (0x9f7adff3 on settle),
#       0 LP claimable while latched and after 7 days, latch persists on flat book
#       (P=2,014,895.20 USDG, L=0, C=0), owner clear restores settlement (989,503.11 shares).

# CRK3b_LatchBlocksTraderWithdraw.t.sol
[PASS] test_CRK3b_LatchBlocksBystanderMarginWithdraw() (gas: 134438955)
  bystander free buying power 7999900000
  before attack: bystander 100 USDG withdraw ok true
  while latched: bystander 100 USDG withdraw ok false   (0x7e9b3b04 = CfdEngine__DegradedMode)
  after loser close: bystander 100 USDG withdraw ok false
  after owner clear: bystander 100 USDG withdraw ok true
Suite result: ok. 1 passed; 0 failed; 0 skipped

# CRK3c_AttackerHeldFreeze.t.sol
[PASS] test_CRK3c_AttackerHeldFullMarginFreeze() (gas: 136825636)
  held SHORT margin locked 1997989382850        (= 1,997,989.38 USDG frozen)
  day 1 owner clearDegradedMode reverted true
  day 7 owner clearDegradedMode reverted true
  day 35 owner clearDegradedMode reverted true
  non-owner clearDegradedMode reverts
  ATTACK carry paid over 35d (6dp) 0
  min owner recap X (6dp) 4395880210            (= 4,395.88 USDG; X-1 still reverts)
  owner cleared after recap X; attacker leg still open size 1997000   (repeatable)
[PASS] test_CRK3c_Control_NormalMarginPaysCarry() (gas: 134435248)
  CONTROL carry paid over 35d (6dp) 8966622314
Suite result: ok. 5 passed; 0 failed; 0 skipped

# CRK3d_NormalMarginHeldFreeze.t.sol
[PASS] test_CRK3d_LpSettlementRevertsDay1And7()
  day 1 settle reverted true          (HousePool__DegradedMode)
  day 7 settle reverted true          (HousePool__DegradedMode)
[PASS] test_CRK3d_NormalMarginHeldLegFreeze()
  held SHORT margin (normal) 67680859957      (= 67,680.86 USDG, no top-up)
  day 1   owner clear reverted after realize true   gap X 4139994348
  day 7   owner clear reverted after realize true   gap X 2604161596
  day 14  owner clear reverted after realize true   gap X 811287255
  day 21  owner clear reverted after realize false  gap X 0
  first day owner clear succeeds 21          (sampled d1,2,3,7,14,21; reverts asserted StillInsolvent)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

PoC files: `packages/perps/test/perps/audit/CRK3_HedgedPairPartialClose.t.sol`, `CRK3b_LatchBlocksTraderWithdraw.t.sol`, `CRK3c_AttackerHeldFreeze.t.sol`, `CRK3d_NormalMarginHeldFreeze.t.sol` (bundled in `plether-CRK3-poc.zip`).

