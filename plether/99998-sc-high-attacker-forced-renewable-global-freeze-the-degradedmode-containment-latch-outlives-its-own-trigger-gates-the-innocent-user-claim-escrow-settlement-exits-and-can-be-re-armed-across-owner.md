# \#99998 [SC-High] Attacker-forced, renewable global freeze: the degradedMode containment latch outlives its own trigger, gates the innocent-user claim/escrow/settlement exits, and can be re-armed across owner rescues

**Submitted on Nov 16th 2026 at 01:38:43 UTC by @OxSCSamurai for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99998
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

Attacker-forced, renewable global freeze: the degradedMode containment latch outlives its own trigger, gates the innocent-user claim/escrow/settlement exits, and can be re-armed across owner rescues

**Program:** Plether Immunefi audit competition · commit `a82d4fef` · scope `packages/perps/src/**`
**Impact class (this program's list):** High — *"Temporary freezing of funds for at least 24 hours"* (competition-page impact list, verbatim). Not claimed: Critical/permanent (a working — if griefable and expensive — owner rescue exists); not insolvency (the documented fail-soft boundary); not theft.

## 1. Summary

An attacker can **manufacture and time** a global freeze of unrelated users' funds — matured LP
redemption escrow, position-holder withdrawals, new opens, LP deposits, and individually affordable
trader claims — for **at least 24 hours** (demonstrated ≥72 h, persisted +26 h and +3 d
variants), with no breach of the documented solvency invariant, at measured net attacker result of
**+$15,940.90** (direction-dependent, §6) or **$0 position capital + ~$0.20 bounty outlay** via a
stale protection-anchor primitive filed as a separate report. The $0-position-capital route depends
on that companion report being accepted; if it is not, that leg reduces to the hedged-book route
already proven in this report. The High freeze impact does not depend on it.

The trigger is the `degradedMode` insolvency latch in `CfdEngine.sol`. The bug is not that the latch
exists — it is documented containment — but a three-piece composition:

1. **The latch is reachable by an ordinary, hedged, self-funded trade.** Admission only requires
   `E ≥ L + B` (`B = 25 bps·L`). A balanced attacker book sits the pool at the boundary; a ~2.2%
   ordinary price drift then lets one routine winner close push `E < L` and latch the flag — at a
   moment of the attacker's choosing (e.g. while a victim's matured LP redemption sits unsettled).
2. **The latch never re-evaluates.** `_enterDegradedModeIfInsolvent` returns early once
   `degradedMode` is set (`CfdEngine.sol:1355-1356`), so after *permissionless* liquidations restore
   `E ≥ L` the flag remains latched. Only the owner can clear (`clearDegradedMode` is `onlyOwner`).
3. **Every gated user exit reads the flag (or a second global gate).** Positioned-account withdrawals,
   opens, LP deposits, LP epoch settlement (the only path that releases matured redeem/deposit
   escrow), and — via a separate `pool ≥ totalClaims` gate (`CfdEngine.sol:224`) — *individually
   affordable* trader claims are all blocked while the flag stands.

The freeze then **survives rescue**: `recapitalizePool` + `clearDegradedMode` are non-atomic, so a
pending winning close executes between them, captures the recapitalization, and makes the clear
revert; and each minimal owner recap + clear can be **re-latched** by another pending attacker
close. The undocumented deltas — not the documented containment itself — carry this report (§7).

**What the freeze does NOT block (verified, stated up front):** flat-account withdrawals, closes,
liquidations, margin top-ups (`depositFor` + `addMargin` are ungated — proven in
`PRESUBMIT2MarginDefense.t.sol`), and already-funded epoch claims all stay live. The harmed set is
the users holding trapped escrow/claims/positions who cannot or do not route around the gated
surfaces — the freeze is real, not absolute.

## 2. Affected code anchors

| Anchor | Role |
|---|---|
| `CfdEngine.sol:1355-1356` | `if (degradedMode) return;` — the latch never re-evaluates once set |
| `CfdEngine.sol:1320, :1344` | invoked after every close and every liquidation |
| `CfdEngine.sol:658-667` | `clearDegradedMode() onlyOwner` — reverts `StillInsolvent` unless `availCash ≥ L` |
| `CfdEngine.sol:632-653` | `recapitalizePool() onlyOwner` — owner's own wallet; non-atomic with clear |
| `CfdEngine.sol:224` | `pool.totalAssets() < totalTraderClaimBalanceAssets → revert` — global all-claims gate |
| `CfdEngine.sol:153` | `settlementBufferBps = 25` — the thin admission buffer |
| `CfdEngine.sol:849-856` | `_payOrRecordTraderClaim` — all-or-nothing fresh-pay vs claim |
| `CfdEngine.sol:501-529` | `addMargin` — verified NO degradedMode check (margin defense stays live) |
| `MarginClearinghouse.sol:371-395` | `depositFor` — verified NO degradedMode check (free-settlement entry stays live) |
| `MarginClearinghouse.sol:573-578` | `lockPositionMargin` — engine-operator check only; NO degradedMode check |
| `CfdEngineSettlementSidecar.sol:261-275` | flat accounts (`pos.size == 0`) return at :272 before the degraded revert at :274 — flat withdrawals confirmed ungated |
| `CfdEngine.sol:882-907` | `isFadWindow` calendar — Fri 16:30→Sun 17:15 NY (~48 h weekly) |
| `script/DeployPerpsArbitrumSepolia.s.sol:434-445` | production risk params (repo-root `script/`) |

## 3. Freeze radius (production-param evidence)

| Surface | Gate | Evidence |
|---|---|---|
| Positioned-account withdrawal | `CfdEngine__DegradedMode` (`checkWithdraw`) | `L17_01`: T3 free settlement **$99,992.20** frozen ≥72 h |
| New position open | `OrderRouter__DegradedMode` | `L17_01`: T6 queued open → status 3 (Failed) |
| New LP deposit | `DepositsUnavailable` | `L17A_05` |
| LP epoch settlement — only release for matured escrow | `HousePool__DegradedMode` | bob's matured **~$1.0 M** junior redeem escrow trapped ≥72 h (`L17A_01`, `L17_01`) |
| Matured-redeem cancel/refund | `RedeemCancellationUnavailable` / `RedeemRefundUnavailable` | `L17_01` |
| Affordable claim | `pool < totalClaims → revert` (:224) | `L17_01`: T5's **$8,996** claim frozen; `L17M_03`: T5's **$37,465** claim frozen while pool holds **$193,348.12** vs claims **$486,147.20** |
| Margin top-up / early close | `addMargin` :501-529 and `depositFor` :371-395 have **no** degraded check; closes live | **verified live by execution** — `PRESUBMIT2MarginDefense.t.sol` 2/2 PASS: a latched victim deposits free settlement and `addMargin`s (margin 3,000,000,000 → 62,917,524,958 atoms) and the identical liquidation call that kills the undefended control reverts `PositionIsSolvent`. Victims *can* defend margin or exit during the freeze; the harmed set is those who don't act before the FAD threshold catches them (`L17M_07`) |

Already-funded epoch claims still pay during the latch (`L17A_09`: $4,112.40 paid) — the trap is
*unsettled/queued* escrow, not funded claims. Flat-account withdrawals are not flag-gated
(`CfdEngineSettlementSidecar.sol:261-273` — the flat branch returns before the degraded check).

## 4. Weaponization (all figures from passing tests at commit a82d4fef)

- **Timed latch:** `L17A_01` — boundary slack **$6,600.90**; drift needed **220 bps = 2.2 %**; firing
  close pays the attacker **$12,228.25** fresh; `degradedMode` latches on routine `Exec` (status 2).
  `L17A_02` control: identical drift on a normal-slack book → no latch, victim paid $90,099.28.
- **Zero-capital route:** `L17A_07` — a victim's own armed protection is triggered using a pre-arm
  signed tick (the root cause of the companion report filed alongside this one); the forced close
  latches the flag. Attacker position capital **$0**; bounty outlay measured **200,000 atoms
  (~$0.20)**. This route is a bonus, not a load-bearing fact of the High claim: if the companion
  report is not accepted, the latch is still reached by the hedged-book path (`L17A_01`).
- **Stale latch:** `L17V_03` — after permissionless loser-liquidations, `E = $903,821.58 ≥ L =
  $300,000` and the flag is **still latched**.
- **Rescue griefing:** `L17V_08` — owner `recapitalizePool($95,361.77)` then `clearDegradedMode()`; a
  pending attacker winner-close executes *between* the calls, captures the recap, and the clear
  reverts `StillInsolvent`. Winner captured **$147,728.69** total.
- **Re-latch treadmill:** `L17V_06` — owner capital burned across two clear cycles: **$243,091.46**,
  pool still insolvent.
- **Durable rescue bound:** `L17V_09` — un-griefable rescue costs **$465,877.15 = 91.68 % of pool,
  4.885× the naive `E−L` gap**.
- **Flag-independent claim gate:** `L17M_05` — flat-book state (`L=0`, `E=0`): owner successfully
  clears the flag, yet T5's **$37,465** claim still reverts `InsufficientPoolLiquidity` (pool
  $193,348.12 < claims $486,147.20). Claims minted **inside** the freeze grew $0 → $486,147.20.
- **Silent redeem intake:** `L17A_05` — `requestRedeem` is *accepted* into the dead queue during the
  latch while `requestDeposit` is hard-gated — the freeze silently grows its victim set.
- **FAD-window amplifier:** `L17M_07` — a victim short solvent at maintenance through the latch has
  liquidation revert `PositionIsSolvent` pre-FAD; after the freeze drags across the Friday boundary
  the same position liquidates at `fadMarginBps = 300` (30× stricter than maint 10). To be precise:
  margin defense is **not** impossible while latched — `depositFor`/`addMargin` are ungated and
  `PRESUBMIT2MarginDefense.t.sol` proves a latched victim adds margin and survives the identical
  liquidation call. The amplifier applies to victims who do not (or cannot afford to) act before
  the FAD threshold catches them.
- **Attacker exits first:** `L17A_06` — on owner clear, the attacker exits instantly via flat-account
  withdraw (the firing close was paid *fresh*, so no claim needs settling) while the victim's matured
  redeem is still unclaimable (`claimable == 0`, escrow intact).

## Link to Proof of Concept

https://gist.github.com/dappconsulting/c1bcbceafc1e0b6e6325a62fdc98f777

## Proof of Concept

## 5. PoC instructions + results

Secret Gist:
> https://gist.github.com/dappconsulting/c1bcbceafc1e0b6e6325a62fdc98f777

All local, production params (`vpiFactor 0.01e18, skew 0.4e18, maint 10, init 20, fad 300, carry 500,
minBounty 1e6, bountyBps 10, keeperShare 5000, protocolShare 0` — verbatim
`DeployPerpsArbitrumSepolia.s.sol:434-445`). `L17RealisticFreeze.t.sol` and the `L17_02` companion
test use the repo's fixture params rather than production params and are offered as companion
evidence.

Seven test files plus one test-only helper accompany this report in the attached gist. To
reproduce, at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

1. Copy the eight files to `packages/perps/test/perps/hunt/` — `L17MFixValidation.t.sol` imports
   `L17MCfdEngineFixed.sol` (a verbatim copy of `CfdEngine` with the §8 fix applied; `src/` is
   untouched and the patch is etched onto the deployed bytecode in-test via `vm.etch`).
2. From `packages/perps`, run:

```
ulimit -v 13000000
flock /tmp/forge.lock forge test --match-path test/perps/hunt/L17VVerify.t.sol -vv
flock /tmp/forge.lock forge test --match-path test/perps/hunt/L17RealisticFreeze.t.sol -vv
flock /tmp/forge.lock forge test --match-path test/perps/hunt/L17ClaimSettleFreeze.t.sol -vv
flock /tmp/forge.lock forge test --match-path test/perps/hunt/L17AForcedLatch.t.sol -vv
flock /tmp/forge.lock forge test --match-path test/perps/hunt/L17MFixValidation.t.sol -vv
flock /tmp/forge.lock forge test --match-path test/perps/hunt/L17MResidual.t.sol -vv
flock /tmp/forge.lock forge test --match-path test/perps/hunt/PRESUBMIT2MarginDefense.t.sol -vv
```

Expected results:

| File | Tests | sha256 |
|---|---|---|
| `L17VVerify.t.sol` | 9 passed; 0 failed | `e2ba61a0226965e9e13a080d7923c368d3dd774b5de1d2ce1b46d8029a7481fb` |
| `L17RealisticFreeze.t.sol` | 4 passed; 0 failed | `0df1869ff9fa059738a2e14f9832579be551c7644bc10cfd987e94f681a5f1df` |
| `L17ClaimSettleFreeze.t.sol` | 3 passed; 0 failed | `35773aa40c7aedabfd54aa2a2c27376e4e1fcf8412979f08563b668e4a9b985c` |
| `L17AForcedLatch.t.sol` | 11 passed; 0 failed | `707ed5483cde298b642ecc5c0aa290ecddb53659809276f33cde6a1a38f887b0` |
| `L17MFixValidation.t.sol` | 4 passed; 0 failed | `e301f0dd173b6434c625b3c33f421549f6be3af1ec621a2cd5dd1dfc629fde2e` |
| `L17MResidual.t.sol` | 5 passed; 0 failed | `17fee5b77bf23505ba72a23f0657767bdbfdc4598aa4699d4638222d27feb6e5` |
| `PRESUBMIT2MarginDefense.t.sol` | 2 passed; 0 failed | `e1f387fdd570addef665aa6217bdceccff5befdd907737a2e32bae751b5d643a` |
| `L17MCfdEngineFixed.sol` (helper, not a test) | — | `e15ec460c132d11b15472af0d4d978567b02dfc09d465978abbb6fc4c0123d35` |

Every dollar figure in this report comes from these tests' logged output and assertions.

## 6. Attacker economics (honest)

$600,000 deposits across 6 funded accounts (only ~$60 k locked as margin; **$611,429.82** recoverable
immediately post-fire + **$4,511.08** residual equity) → measured net attacker result **+$15,940.90**
in this run — i.e. the attacker ended ahead; the net includes post-clear claim value and directional
exposure, so it is direction-dependent and not guaranteed. The independent route needs **$0**
position capital (the companion report's primitive). Profit is not required: the freeze of innocent funds is the
impact. Fixture-param companion `L17_02`: full unfreeze recap bound **$2,931,066 = 38.06× the seeded
test pool** (test's own logged ratio).

## 7. Intended-design analysis — what is documented vs what is new

Conceded verbatim (the documented containment, not claimed):

| Doc | Verbatim | Status |
|---|---|---|
| ACCOUNTING_SPEC.md:1201-1238 | `degradedMode` = "containment latch, not a retroactive revert"; blocked: "new opens, other risk-increasing modifications, trader withdrawals while the account has a live position, new LP epoch settlement, including redemption funding and deposit activation"; "Already-funded LP claims remain claimable"; owner clear at `E ≥ L` | latch + most of the freeze radius documented |
| ACCOUNTING_SPEC.md:619-620 | "sticky on-chain latch… only an authorized `clearDegradedMode()` call" clears | stickiness documented |
| WHITEPAPER.md:1187-1188 | "An account claim may be settled only by its beneficiary and only when aggregate claims are fully covered" | all-or-nothing gate documented |
| SECURITY.md:340 | degraded containment "blocks further risk expansion while still permitting protective transitions" | documented — but silent on exits being bricked |
| SECURITY.md:942-943 | latch "may be cleared at E >= L" | documents the *ability* to clear — not that restored solvency does nothing, that clearing is owner-exclusive, or that the clear is griefable |
| CfdEngine.sol:725-726 (NatSpec) | "Open positions require non-degraded mode" — the positioned-withdraw gate is disclosed | conceded: the gate itself is documented; the deltas below are not |
| SECURITY.md:500-502 | in the *guardian-authority* section — guardian powers exclude "a global protocol freeze", i.e. no admin can impose one | read precisely: the doc does **not** promise no freeze can happen — it grants no actor that power. Yet an ordinary close produces a de-facto freeze of exit paths (positioned withdrawals, matured LP escrow, epoch settle, individually affordable claims) that even the guardian could not impose, lasting ≥24 h with owner-only release. Impact scope exceeds documented containment; the latch itself is NOT claimed as a bug |
| SECURITY.md:157-162, 1499-1511 | owner recapitalization + terminal-deficit runbook | documents the steps; omits that they are griefable mid-sequence and that a durable un-griefable rescue costs ~91.7 % of pool |

**The undocumented deltas this report claims (none appears in any program document or audit):**

1. **Griefable non-atomic rescue** — a pending close between `recapitalizePool` and
   `clearDegradedMode` captures the recap and re-fails `E ≥ L` (measured 95.4 k recap → 147.7 k
   winner capture; clear reverted).
2. **Re-latch treadmill** — queued winner closes re-latch after each minimal recap; durable rescue
   ≈ 91.7 % of pool rather than the gap.
3. **Silent intake asymmetry** — redemption *requests* are accepted into a queue that cannot settle
   while `requestDeposit` is hard-gated.
4. **Attacker-scheduled timing** — the latch is fireable at a chosen boundary moment (~zero cost),
   converting a documented safety state into a trap for specific pending user funds.
5. **Flag-independent claim gate** — claims stay frozen after the flag clears (WHITEPAPER documents
   the gate; nothing documents that an attacker can push the system across it on demand).

**Nearest documented items and why they do not cover this report:** the closest design rationale
for a similar end-state rests on (a) the attacker's cost exceeding the victims' damage and (b) the
end-state being documented design. This route removes (a) — the victims hold the claims while the
caller profits in the measured run — and (b) is addressed above: the latch itself is conceded, the
five deltas are not documented anywhere. Prior audits (Cecuro, SC Audit Studio, the collaborative
audit) and the program's own known-limitation list contain no finding covering rescue griefing,
re-latch, redeem-intake asymmetry, scheduled latch, or the post-clear claim gate.

## 8. Recommended fix + fix-validation evidence

1. Auto-clear (or permissionless clear) when `E ≥ L` — remove the early return; the `E ≥ L` check
   already makes a false clear impossible.
2. Per-claim gate `claim ≤ pool − (L + B)` — preserves ordering fairness without hostaging
   affordable claims.
3. Optionally make `recapitalizePool + clear` atomic.

`L17MFixValidation.t.sol` (4/4 PASS) etches a verbatim-patched engine (`L17MCfdEngineFixed.sol`,
test-only; `src/` untouched) onto the same frozen state: recovery auto-clears, positioned withdraws
resume, opens admitted, LP epoch settles, the $37,465 claim pays under the per-claim gate while the
$224,341 over-claim still correctly reverts, and the latch still stands at `E < L` (containment
preserved — the fix removes overshoot, not the documented feature).

## 9. Scope check

All contracts in path are in scope (`packages/perps/src/**`). No Sybil dependence — SECURITY.md:352
(single direction per account) *requires* ≥2 accounts for a hedged book; every position is real
funded capital held by six separately funded accounts; no identity limit is bypassed. No
trusted-role misbehaviour — owner recap/clear are documented trusted powers exercised *correctly* in
the PoC; the harm is that the correct sequence can be griefed. Keepers are untrusted by program
rules — permissionless callers are in scope.

## 10. Severity

**High — Temporary freezing of funds for at least 24 hours.** Demonstrated ≥72 h against a passive
owner; ≥24 h trivially even with a responsive owner (non-atomic, griefable, re-latchable rescue).
Not Critical: an expensive owner rescue exists → temporary, not permanent. Feasibility ($600 k
committed, measured net +$15.9 k in this run, or $0 position capital via the companion report if
it is accepted) affects payout, not the impact class. The High freeze is independently reached by
the hedged-book route.

**Honest bound on the victim set:** the freeze does not hold users who keep
nothing on the gated surfaces — flat-account holders withdraw freely, already-funded claims pay,
and positioned victims who act early can add margin or close. The guaranteed-frozen set is matured
LP escrow, queued/unsettled requests, blocked claims, and positioned accounts caught before they
exit — plus everyone the silent redeem intake traps during the window. The ≥24 h freeze of those
funds is unconditional.
