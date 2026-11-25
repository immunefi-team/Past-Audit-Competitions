# \#98553 [SC-High] Hedged-pair degraded latch freezes LP deposit escrow for over 24h

**Submitted on Nov 8th 2026 at 16:47:26 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98553
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

**The latch.** Degraded mode latches when a close leaves `E < L`, where `E` is pool assets minus trader claims and `L` is the larger side's maximum-profit envelope (`CfdEngine.sol:1351-1368`). While it is latched:
- opens are blocked;
- LP settlement reverts on both routes;
- deposit activation is blocked;
- **matured deposit escrows cannot be cancelled**: `TrancheVault.sol:851-860` reverts with `DepositEpochAlreadyActive` for Junior and `SeniorDepositCommitted` for Senior.

The owner can clear the latch only once `E ≥ L` (`CfdEngine.sol:658-667`).

**How an unprivileged trader gets there.**
1. Open a hedged LONG and SHORT at the admission boundary, each leg ≈ `E − B`.
2. Wait for any move of 25 bps or more.
3. Close the winning leg.

The payout lowers `E` below the surviving leg's `L`.

**Nothing restores `E` without the owner's money:**
- opens are blocked;
- execution fees go to the treasury account, not to the pool;
- carry is collected only when the attacker touches its own position;
- third-party closes help only if they are on the loser's side and large enough.

In the PoC the gap stayed byte-identical for **30 simulated days** with only permissionless calls, and `clearDegradedMode` reverted `StillInsolvent` at every 6-hour probe.

**Victims.** LPs whose deposit epoch matured, or whose Senior reservation is committed, when the latch fires. In the PoC, **100,000 USDG** could be neither cancelled nor activated for the whole period, while the same build without the latch activates both deposits.

**Cost to the attacker.** Fees plus optional carry. Its capital is locked but not lost, and the remaining leg can be closed at any time.

**The owner's remedy.** `recapitalizePool` with 9.6k-37k of its own funds, then `clearDegradedMode` and `assignUnassignedAssets`.
- It is a power, not a documented duty, with no timeline in the runbook.
- A second pre-opened pair undoes it: one capital outlay produced **7 latches across 6 owner recapitalizations** totalling 16,288 USDG.

## Vulnerability Details

Line numbers refer to `immunefi-team/audit-comp-plether` at a82d4fe. The in-scope code is identical to the Arbitrum Sepolia v2.0.0 release.

### 1. The latch fires after any 25 bps move against a pair sized at capacity

```solidity
// packages/perps/src/CfdEngine.sol:1351-1368
function _enterDegradedModeIfInsolvent(address account, uint256 pendingPoolPayoutAssets) internal {
    if (degradedMode) return;
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    ... // minus the pending payout
    uint256 maxLiabilityAssets = _maxLiability();          // L = max(longMaxProfit, shortMaxProfit)
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}
```

Admission only requires `E ≥ L + B` at open (`CfdEnginePlanLib.sol:421` → `SolvencyAccountingLib.sol:93-104`), with `B = 25 bps · L`. A hedged pair sized at capacity therefore leaves exactly `B` of slack, and any winning payout above `B`, which a 25 bps move produces, latches.

### 2. There is no organic recovery

`E` rises only with pool inflows, and while the latch holds each of them is closed off or controlled by the attacker:

| Inflow | Available while latched? |
|---|---|
| VPI charges | No: opens are blocked |
| Frozen spreads, liquidation LP fees | Only if someone closes or is liquidated |
| Carry | Only when the position's owner acts (`realizeCarryBeforeMarginChange`, close) |
| Trader losses | Only from loser-side closes |
| Execution fees | No: credited to the treasury's clearinghouse account (`CfdEngineSettlementSidecar.sol:446-463`), not to pool assets |

Measured over 30 days:
- at +2%, the gap stayed at 9,588.73 USDG;
- at +5%, the gap was 37,374 and was still 17,140 after every unrelated trader closed.

### 3. Matured deposit escrows can be neither cancelled nor activated

```solidity
// packages/perps/src/TrancheVault.sol:851-860 (cancelPendingDeposit)
if (
    (IS_SENIOR || currentLpEpoch() >= requestId) && !queueState.rejected
        && (!IS_SENIOR || !POOL.lpEpochSettlementPaused()) && !_projectedTerminalWipe()
        && !POOL.isSeniorImpairedAfterPendingDepositReconcile()
) {
    if (IS_SENIOR) revert TrancheVault__SeniorDepositCommitted();
    revert TrancheVault__DepositEpochAlreadyActive();      // degraded mode is not an escape condition
}
```

- Activation is blocked: `settleLpEpoch` reverts `HousePool__DegradedMode` on both routes (`HousePool.sol:832/:1868`), and so does the LP entry gate (`HousePoolTrancheGateLib.sol:28-38`).
- The pending deposits would themselves restore `E ≥ L` if activated, but activation is exactly what the latch blocks.
- SECURITY.md:493-495 lists the conditions that open a matured deposit's cancellation: rejection, projected wipe, Senior impairment and the explicit settlement hold. Degraded mode is not among them, although it blocks activation exactly like the hold.

### 4. Only the owner can end it, and a second pair re-latches

- `clearDegradedMode` requires `E ≥ L` (`CfdEngine.sol:662`). The only lever is `recapitalizePool` from the owner's wallet, followed by `assignUnassignedAssets` (`HousePool.sol:660`).
- With a second hedged pair pre-opened before the first latch, each 50k slice of the second winner re-latches with a gap of about 1.0-1.15k after each recovery. 7 latches were measured.

### 5. Escape routes for the victims

| Victim | Way out |
|---|---|
| Junior depositor | **None**: `rejected` is set only inside the settlement that degraded mode blocks; pause does not unlock |
| Senior depositor | Cancel only if the owner enables the explicit settlement hold |
| Matured redeem request | Cannot be cancelled (`TrancheVault.sol:894-896`, `RedeemCancellationUnavailable`); shares stay in non-transferable vault escrow, and funding is blocked by the latch (not counted in the frozen amount) |
| Positioned trader | Can full-close, then withdraw |

### Relation to prior audits and known issues

- **What is documented.** The degraded latch itself, and the fact that degraded mode blocks LP entry and settlement: ACCOUNTING_SPEC.md:1201-1230; EMERGENCY_RESPONSE_GUIDE.md:45-47; SECURITY.md:340, :1504-1510.
- **What is not documented:**
  - that an unprivileged trader can create the condition at will, direction-neutrally, and hold it with zero permissionless recovery;
  - that a Junior depositor then has no exit path at all;
  - any timeline or duty for the owner's recapitalization.
- **Prior audit items** on degraded mode (MED-18 equality cases, LOW-73 protection in degraded mode) concern other paths.
- **Not "lack of liquidity".** The frozen asset is the depositor's **own escrow** held in the vault, not a claim on pool cash. Activating it would increase pool assets. The freeze comes from the latch blocking activation and cancellation, not from missing cash.

## Impact Details

**High, "Temporary freezing of funds for at least 24 hours".** It is claimed on the LP deposit escrow: every matured or committed deposit at the time of the latch, sustained by the attacker for 30+ days unless the owner injects cash. The program assumes no detection or response deadline for trusted roles, so the measured duration without owner action is the relevant one.

| Measurement | Value |
|---|---|
| Frozen deposit escrow (PoC pool: Junior 210k, Senior 750k) | **100,000 USDG for 30 days**, versus activation in the control |
| Same at 10M pool (Junior 2M, Senior 8M), `W7Senior.t.sol` | **1,000,000 USDG frozen for 30 days**, plus 500k of matured redeem requests unfunded |
| Attacker cost per latch cycle | about 758 USDG of fees plus about 6k VPI round trip, plus 0-83 USDG/day carry |
| Attacker cost at 10M (net of an 80% Junior stake) | 31.4k, about 1,045 USDG per day |
| Damage/cost on frozen principal at 10M | 48× (18× without an LP stake, 173× at the fee floor) |
| Owner recapitalizations undone by a second pair | 7 latches across 6 recapitalizations (16,288 USDG injected) |

**Who gains.** The attacker gains nothing at the victims' expense: this is a freeze without extraction.
- The Senior coupon only changes its booking date.
- Liquidations and margin top-ups still work for positioned victims.
- The owner's recapitalization comes back to the owner as Junior shares (9,588.73 in, 9,677.26 out).

**Capital.** The capital locked by the attacker is about 1.2M, well within the program's rule that capital below USD 100M is not a reason to downgrade.

### Preconditions

- **Pool.** At or near admission capacity for one trader's pair at release params. In the PoC: pool Junior 210k plus Senior 750k, pair 936k per leg. This scales with pool size, since the slack is always `B = 25 bps · L`.
- **Victims.** Any LP whose deposit epoch matured, or whose Senior reservation is committed, when the latch fires. Deposits are requested hourly, and the attacker can time the winner close right after a matured epoch's cutoff.
- **No trusted role, no oracle trust, no third-party failure.** All calls are public: `commitOrder`, `executeOrder`, `addMargin`.

## Recommendation

Do one of the following:
- Treat degraded mode like the explicit settlement hold for escrow refunds: allow `cancelPendingDeposit` for matured Junior and committed Senior deposits while `degradedMode` is latched.
- Allow deposit activation during degraded mode when the activation itself restores `E ≥ L`.

The gist's `fix.diff` takes the first option: `cancelPendingDeposit` also opens while the engine is degraded (one condition plus a view helper in `TrancheVault.sol`).
- Against the fixed copy, the freeze test fails because both victims recover their 50k deposits while the latch is still on.
- 622 of 624 repo LP tests pass. The two failures are the repo's 24,000-byte size-budget test and a pre-existing gas test; the project may prefer exposing the flag through an existing pool view.

## Link to Proof of Concept

https://gist.github.com/the-red-ace/dc29345d1718ab9b9b7f1dd923fd3110

## Proof of Concept

## Proof of Concept

**Gist:** https://gist.github.com/the-red-ace/dc29345d1718ab9b9b7f1dd923fd3110. It contains the test files, `fix.diff` and a README with the exact folder for each file.

**How the tests run:** in the repo's own Foundry suite at a82d4fe, with release parameters. No in-scope contract is modified. The only mocks are Pyth price updates and mock USDG, both allowed by the program.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/{w3,g1,x2,w7}
cp W3Degraded.t.sol packages/perps/test/perps/w3/
cp G1Base.sol packages/perps/test/perps/g1/
cp X2GridBase.sol X2ErrNames.sol packages/perps/test/perps/x2/
cp W7Senior.t.sol packages/perps/test/perps/w7/
forge test --root packages/perps --match-path test/perps/w3/W3Degraded.t.sol --no-match-test test_fix -vv
forge test --root packages/perps --match-path test/perps/w7/W7Senior.t.sol --no-match-test test_fix -vv   # 10M pool scale
```

| Test | What it asserts |
|---|---|
| `test_T1_T2_organicRecovery_noOwner` | 30 days at +2% and +5% with only permissionless calls; the gap does not shrink, and `clearDegradedMode` reverts `StillInsolvent` at every probe. |
| `test_T1b_thirdPartyFlows`, `test_T1c_bigMove_allThirdPartiesExit` | Third-party trading and exits do not restore `E ≥ L`. |
| `test_T3_attackerCost` | The attacker's cost per latch cycle. |
| `test_T4_relatch` | A second pre-opened pair re-latches after each owner recapitalization (7 latches, 6 recapitalizations). |
| `test_T5_escapes` | Junior escrow: cancel and activation revert; Senior: only the owner's settlement hold opens cancel. |
| `test_control_noLatch` | Control: the same deposits activate normally without the latch. |
| `W7Senior` | 10M pool: 1M of escrow frozen for 30 days, cost accounting, and no second-role extraction. |

Expected output (reproduced on a clean a82d4fe checkout with only the gist files):
```
Suite result: ok. 7 passed; 0 failed; 0 skipped      (W3Degraded)
Suite result: ok. 8 / 9 / 8 passed; 0 failed         (W7Senior: three scale contracts, each with the 7 inherited W3 tests minus test_fix, plus its own)
```
`test_fix_*` is excluded on purpose: it asserts the behaviour after `fix.diff` is applied, so it fails on the unpatched code by design.
