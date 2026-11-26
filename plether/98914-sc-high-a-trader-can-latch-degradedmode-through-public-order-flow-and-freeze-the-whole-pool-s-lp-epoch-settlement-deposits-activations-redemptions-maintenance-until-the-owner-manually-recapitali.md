# \#98914 [SC-High] A trader can latch degradedMode through public order flow and freeze the whole pool's LP epoch settlement (deposits, activations, redemptions, maintenance) until the owner manually recapitalizes

**Submitted on Nov 10th 2026 at 22:08:03 UTC by @dxhod for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98914
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

A single trader (two EOAs plus a self-run keeper) can push the engine into a genuine raw-insolvency state (`E < L`) using only public order flow, which latches `degradedMode`. `HousePool._settleLpEpoch` checks `degradedMode` before anything else, so **every** LP epoch settlement reverts while the latch is up: deposit activations, redemption funding, and coupon/maintenance checkpoints — for the whole pool, not just one request. The latch is sticky and clears **only** by an owner `recapitalizePool` + `clearDegradedMode`; there is no permissionless and no timelocked recovery, and `clearDegradedMode` reverts `CfdEngine__StillInsolvent` while the attacker keeps the losing leg over-margined (so keepers cannot liquidate it away). No value is stolen — the attacker's spend and the owner recap both flow into the pool — so this is **temporary freezing of LP funds**, not theft.

I claim **Medium — "Temporary freezing of funds for at least 1 hour"**. I deliberately do **not** claim High ("... for at least 24 hours"): recovery (`recapitalizePool` + `clearDegradedMode`) is an immediate, non-timelocked owner action, so I cannot prove the freeze reliably lasts >= 24h against a monitored owner. The latch is sticky with no permissionless and no timelocked recovery, so a floor of >= 1h (detect -> source external USDG equal to the recap gap -> send the recovery txs -> re-open entries) is well supported.

## Vulnerability Details

**Root cause: whole-pool LP settlement liveness is gated on a single sticky latch that untrusted order flow can set, with recovery available only to a non-timelocked owner.**

1. **Build a balanced hedged book to the admission limit, through the public router.** Two attacker EOAs open a LONG N and a SHORT N position in interleaved chunks (each <= 20% of depth, to respect `maxSkewRatio` 0.4), self-executed by the attacker's own keeper via `OrderRouter.commitOrder` + `executeOrder`. The public commit-time admission check stops the fill at the settlement-buffer limit, where the larger side's maximum liability `L` sits just under effective equity `E = totalAssets - traderClaims`. Margin lives in the clearinghouse, not the pool, so it does not move `E` or `L`.

2. **Wait for an ordinary basket move, then close only the winning leg.** After an ordinary ~1% FX move (not oracle-manipulated), the attacker closes **only** the winning leg through the public router. The pool pays the winner's profit, dropping `E` below the untouched losing leg's `L`. `_applyClose` (`CfdEngine.sol:1306,1320`) calls `_enterDegradedModeIfInsolvent` (`CfdEngine.sol:1351-1369`), which sets `degradedMode = true` because `effectiveAssetsAfter < maxLiabilityAssets`. (The move needed is ~1%, not the naive 0.25% buffer, because the VPI charge incurred when the close unbalances the hedge flows back to the pool and offsets part of the winner payout. ~1% is an ordinary FX day; the test sizes the minimum from live state.)

3. **The latch freezes the whole pool's LP settlement.** `HousePool._settleLpEpoch` (`HousePool.sol:824`) calls `_requireWithdrawalsLive` (`HousePool.sol:832`) first; `_requireWithdrawalsLive` (`HousePool.sol:1864-1870`) reverts `HousePool__DegradedMode` on `statusSnapshot.degradedMode` **alone**, before any liquidity or freshness logic. So the public `OrderRouter.settleLpEpoch(bytes[])` and the direct `pool.settleLpEpoch(mark,time)` both revert. Matured requests cannot settle, and they also cannot be cancelled (the vault treats the epoch as active/committed): `cancelPendingDeposit` / `cancelRedeemRequest` revert.

4. **Recovery is owner-only, sticky, and attacker-sustained.** `recapitalizePool` (`CfdEngine.sol:632`) and `clearDegradedMode` (`CfdEngine.sol:658`) are both `onlyOwner`, with no timelock and no permissionless path. While the attacker keeps the losing leg over-margined (keepers cannot liquidate it), `L > E` persists and `clearDegradedMode` reverts `CfdEngine__StillInsolvent` (`CfdEngine.sol:663`). The owner must source external USDG equal to the gap and recapitalize to lift the latch.

5. **Re-trigger.** After the owner clears once and the book is unwound, the attacker can re-fund, rebuild the hedged book through the same public path, and latch again with another ordinary move. Demonstrated at the 1M pool scale (`re-latch after one owner clear: true`). The owner must recapitalize on every cycle.

**Affected contracts (all on the program's in-scope `packages/perps/src` list, verified at commit a82d4fe):**

- `HousePool.sol` — `_settleLpEpoch` (824) -> `_requireWithdrawalsLive` (832); `_requireWithdrawalsLive` (1864-1870) reverts `HousePool__DegradedMode` on `degradedMode` alone (1867-1868). The frozen entry point.
- `CfdEngine.sol` — `_applyClose` (1306) -> `_enterDegradedModeIfInsolvent` (1320); the latch (1351-1369). Recovery `recapitalizePool` (632, `onlyOwner`) and `clearDegradedMode` (658, `onlyOwner`), which reverts `CfdEngine__StillInsolvent` (663).
- `OrderRouter.sol` — `commitOrder` / `executeOrder` build and close the book; `settleLpEpoch(bytes[])` (479, "Permissionless") is the frozen public settle entry.
- `TrancheVault.sol` — matured requests cannot be cancelled while latched: `cancelPendingDeposit` reverts `TrancheVault__DepositEpochAlreadyActive` (859) / `TrancheVault__SeniorDepositCommitted` (857); `cancelRedeemRequest` reverts `TrancheVault__RedeemCancellationUnavailable` (896).
- `MarginClearinghouse.sol` — holds trader margin used to keep the losing leg over-margined so keepers cannot liquidate it (this sustains `L > E`).

## Impact Details

While latched, the whole pool's LP entry/exit pipeline is halted: deposit activations, redemption funding, and carry/maintenance checkpoints all run through the same gated `_settleLpEpoch`. No single request can proceed, and none can be cancelled. Cumulative blocked LP demand grows for the entire freeze.

Numbers are from the runnable PoC at the release parameters (vpiFactor 0.01e18, maxSkew 0.4, maint 10 / init 20 / fad 300 bps, execution fee 4 bps, settlement buffer 25 bps, Senior cap 40M / 80%), 6-decimal USDG, at three pool sizes. `e6` = USDG.

### MEASURED (asserted in the test / printed in the log)

| metric | 1M | 10M | 40M |
| --- | --- | --- | --- |
| public opens to reach the limit | 30 | 28 | 28 |
| `L` pre-attack (max liability) | 998,000.00 | 9,960,000.00 | 39,840,000.00 |
| `E` pre-attack (effective equity) | 1,002,000.22 | 10,002,002.15 | 40,002,008.58 |
| buffer headroom `E - L` | 4,000.22 | 42,002.15 | 162,008.58 |
| winner profit paid by pool on the latching close | 5,878.80 | 60,651.92 | 238,634.88 |
| `E` after close | 996,121.41 | 9,941,350.22 | 39,763,373.70 |
| `L` after close | 998,000.00 | 9,960,000.00 | 39,840,000.00 |
| owner recap to clear (external USDG injected) | 1,878.59 | 18,649.78 | 76,626.30 |
| attacker net cost per latch | 6,251.92 | 59,894.66 | 239,560.90 |
| matured Senior commitment (2%) blocked | 20,000.00 | 200,000.00 | 800,000.00 |
| matured Junior deposit (1%) blocked | 10,000.00 | 100,000.00 | 400,000.00 |
| matured Junior redemption blocked | 1,000.00 | 10,000.00 | 40,000.01 |
| concrete blocked LP demand (sum) | 31,000.00 | 310,000.00 | 1,240,000.01 |
| whole-pool LP settlement halted (pool TVL) | 1,002,000.00 | 10,002,000.00 | 40,002,000.00 |
| second latch after one owner clear | yes | not attempted | not attempted |

The freeze is also asserted to persist across 1h / 24h / 48h / 72h warps (the latch is sticky absent an owner action). The re-latch was passed `true` only for the 1M run; the 10M/40M runs were executed with re-trigger disabled, so "not attempted" there is not a refutation.

### DERIVED (computed from the measured numbers, not separately asserted)

- Attacker net cost ~= 0.6% of pool TVL, dominated by the VPI charge on the unbalancing winning-leg close (the hedge's FX nets out because both legs close at the same moved price; the residual is fees + VPI + carry). It scales ~linearly with pool size.
- Attacker cost ~= 3.1-3.3x the external USDG it forces the owner to inject (cost / recap = 3.33 / 3.21 / 3.13). Pure griefing: zero attacker profit.
- The attacker also fronts ~pool/2 of margin (capital/4 per leg), most of which is returned on unwind.

### Griefing-ratio rule and the "lack-of-liquidity" exclusion

**Griefing ratio.** The program rule: $1 spent for <= $10 damage = Griefing; for >= $100 damage = not griefing.

- Against the concrete blocked escrow (Senior 2% + Junior 1% + the headroom-sized redeem): damage / cost ~= 5x (MEASURED ratio `blockedDemand / attackerCost` = 4 / 5 / 5). Griefing band.
- Against the whole-pool settlement TVL that is halted: damage / cost ~= 160-167x (DERIVED). Not-griefing "Temporary freezing of funds" band.

Either way the severity lands at **Medium** (Medium "Griefing" or Medium "Temporary freezing of funds for at least 1 hour"). The whole-pool halt is the operative effect: all LP entry/exit is frozen and cumulative blocked demand grows for the whole freeze.

**"Lack-of-liquidity impacts" exclusion does not apply.** The PoC isolates the latch from any liquidity shortfall: after latching, the attacker closes the losing leg too, leaving a **flat book (`L = 0`), a solvent pool (`E >> 0`), and full free cash** — yet `OrderRouter.settleLpEpoch` **still** reverts `HousePool__DegradedMode`. `_requireWithdrawalsLive` (`HousePool.sol:1864-1870`) gates on `degradedMode` alone, independent of liquidity, and nothing permissionless clears it. So a fully solvent, fully liquid, position-free pool stays frozen purely by the sticky latch until the owner acts.

**No value is lost.** The attacker's spend and the owner recap both flow into the pool (they enrich LPs). The harm is temporary illiquidity of LP funds enforced by a sticky, owner-only-clearable latch — not theft or insolvency.

### Known-issue / by-design analysis (disclosed honestly)

**Degraded mode is documented as intended containment.** The mechanism and the owner recovery runbook appear in the design/ops docs: the engine latches `degradedMode` when a close or liquidation leaves raw `E < L`, it blocks new risk and LP settlement, and the owner recapitalizes or clears positions and then clears degraded mode once raw `E >= L`. A judge can reasonably consider "containment working as designed."

**But by the program's own narrow known-issue definition, this specific finding is not a known issue.** The program scopes known issues to (a) the three listed audits (incl. accepted/acknowledged), (b) the SECURITY.md known-limitation sections, and (c) the explicitly listed open items:

- **SECURITY.md "## Known Limitations" (lines 1328-1457):** a case-insensitive search for "degraded" returns hits at lines 27, 112, 161, 246, 340, 542-565, 663, 938-942, 1324, 1506-1510 — **none inside 1328-1457**. The designated Known Limitations block has zero mention of degraded mode. The documentation that describes it is the *design narrative* (SECURITY.md:340) and the *operator runbook* (SECURITY.md:1506-1510 / EMERGENCY_RESPONSE_GUIDE.md:45-47), not the Known Limitations list.
- **The three listed audits:** degraded mode appears only in unrelated framings (e.g. a trader cannot add stop-loss protection while degraded is active [Acknowledged]; views misreporting during degraded; a note that `assignUnassignedAssets` does not clear degraded; optimistic liveness views). **None** describes a trader deliberately building a hedged book and closing the winner to force `E < L` and freeze whole-pool LP settlement. No literal match of root cause + impact.
- **Listed open items:** none concern degraded mode.

So the novel, undocumented part is the **adversarial build-sustain-grief** path (a trader engineers and sustains the insolvency through untrusted order flow) and the **whole-pool settlement halt** with **no permissionless or timelocked recovery**. The documented firewall is narrower (redemption funding), whereas `_requireWithdrawalsLive` halts the entire epoch. I state plainly that the insolvency is genuine (CAP_PRICE is a conservative 2x worst case), so degraded mode is firing correctly; the claim is not "the latch is a bug" but "untrusted order flow can deliberately and repeatably trigger and sustain it to freeze all LP liveness, with recovery available only to the owner and not timelocked."

**Trusted-role and parameter exclusions do not apply to the trigger.** Building the book, closing the winner, holding the loser over-margined, and re-triggering are all untrusted public-path actions ("Keepers are not trusted"). The trusted-owner action (`recapitalizePool` + `clearDegradedMode`) is the recovery side — the victim's remedy — not the trigger. Release parameters are used; no owner parameter change is needed.

## References

- `packages/perps/src/HousePool.sol:824,832,1864-1870` — `_settleLpEpoch` -> `_requireWithdrawalsLive`; reverts `HousePool__DegradedMode` on `degradedMode` alone
- `packages/perps/src/CfdEngine.sol:1306,1320,1351-1369` — `_applyClose` -> `_enterDegradedModeIfInsolvent` latches `degradedMode` when `effectiveAssetsAfter < maxLiabilityAssets`
- `packages/perps/src/CfdEngine.sol:632,658,663` — `recapitalizePool` / `clearDegradedMode` (`onlyOwner`, no timelock); `clearDegradedMode` reverts `CfdEngine__StillInsolvent`
- `packages/perps/src/OrderRouter.sol:479` — permissionless `settleLpEpoch(bytes[])` (frozen entry point); `commitOrder` / `executeOrder` build and close the book
- `packages/perps/src/TrancheVault.sol:857,859,896` — cancel paths revert while latched
- Error selectors: `IHousePool.HousePool__DegradedMode` (0x9f7adff3, `interfaces/IHousePool.sol:136`); `ICfdEngineTypes.CfdEngine__StillInsolvent` (`interfaces/ICfdEngineTypes.sol:65`)
- Commit a82d4fefb02768e5b3bddcc97dee371eee24fea8


## Link to Proof of Concept

https://gist.github.com/dxhod/0e57323c01688d423c02e21da3029cda

## Proof of Concept

## Proof of Concept

Self-contained Foundry test against the competition commit `a82d4fe`, using the repo's own `BasePerpTest` harness at the release parameters. Only USDG and Pyth are mocked (the allowed mocks).

**Every attack-critical step is a call from an attacker EOA to a public entry point:** the hedged book is built ONLY through `OrderRouter.commitOrder` + `executeOrder` (self-executed by the attacker's own keeper); the latching close is the public router close; the freeze is shown on the public `OrderRouter.settleLpEpoch(bytes[])` and the direct `pool.settleLpEpoch`; cancels are the public TrancheVault paths. The owner-recovery steps (the victim side) run as the owner. The orchestration lives in a library (`H03V2Lib.sol`) run via delegatecall, so `address(this)` is the test contract = the engine/pool owner; untrusted steps are `vm.prank`-ed from attacker EOAs. The library split is only to keep the test contract under the via-IR tag limit; it does not change who calls what.

### Steps to reproduce

1. Place both files below under `packages/perps/test/sca/`: `H03V2.t.sol` and `H03V2Lib.sol`.
2. From the repository root of the local copy, run:

```sh
export PATH="$HOME/.foundry/bin:$PATH"
forge test --root packages/perps --match-path test/sca/H03V2.t.sol -vv \
  --out out-sca-H03V2 --cache-path cache-sca-H03V2
```

(If `out-sca-H03V2` / `cache-sca-H03V2` do not exist, seed them once with `cp -R packages/perps/out packages/perps/out-sca-H03V2` and the same for `cache`, otherwise the via-IR build runs from scratch (~18 min).)

### Result

`3 passed` — `test_H03V2_scale_1M` / `_10M` / `_40M`.

### Key assertions proved by the test (not just logged)

- Pre-attack solvent (`E > L`), not degraded, both legs built through the public path.
- CONTROL: with degraded mode off, the identical matured book settles — Senior commitment and Junior deposit activate and the headroom-sized Junior redemption funds through the public `settleLpEpoch`. This proves ordinary LP liveness holds on the same book, so the freeze is the latch, not a generic liquidity limit.
- ATTACK: after the public winning-leg close, `engine.degradedMode() == true` and raw `E < L`; the winner leg is fully closed and the loser leg is held open.
- FREEZE: the public `OrderRouter.settleLpEpoch(bytes[])` and the direct `pool.settleLpEpoch(mark,time)` both revert `HousePool__DegradedMode` (0x9f7adff3). Matured Senior/Junior deposits and the Junior redemption have `claimable == 0` and their cancels revert. Re-checked after 1h / 24h / 48h / 72h.
- OWNER CLEAR BLOCKED: `clearDegradedMode()` reverts `CfdEngine__StillInsolvent` while the attacker holds the over-margined loser leg.
- ISOLATION: close the loser too -> flat book (`L == 0`), solvent (`E > 0`), full cash — yet `settleLpEpoch` STILL reverts `HousePool__DegradedMode`. Proves the freeze is the sticky latch, not lack of liquidity.
- RECOVERY: owner `recapitalizePool(gap)` + `clearDegradedMode()` lifts the latch; afterwards `settleLpEpoch` no longer reverts `HousePool__DegradedMode`.
- RE-TRIGGER (1M): a second latch is reached after one owner clear (`re-latch == true`).

### Full runnable test code

The two Foundry files (`H03V2.t.sol`, `H03V2Lib.sol`) that produce the result above are provided with this report (see the linked secret Gist / attachment). Place both under `packages/perps/test/sca/` and run the command above.

**Secret Gist with both runnable files:** https://gist.github.com/dxhod/0e57323c01688d423c02e21da3029cda

