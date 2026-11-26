# \#98754 [SC-High] Near-zero Junior share inflation makes every `settleLpEpoch` overflow and freezes Senior redemptions for days

**Submitted on Nov 10th 2026 at 02:19:49 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98754
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

**The gate.** `HousePool.settleLpEpoch` prices Junior deposits at `shares = net * (S + 1000) / (P + 1)`. Here `S` is the Junior share supply and `P` is the reconciled `juniorPrincipal`. Activation is deferred only when `P == 0` **exactly** (`HousePool.sol:917`).

**The inflation.** The Senior coupon is paid out of Junior principal continuously, so a Junior on its way to wipe passes through every small value at a predictable time. A deposit `D` settled while `P` is a few thousand atoms multiplies `S` by about `D/P` (~3,000x). The coupon then drains `D` back to dust in about 3 hours. After 17 such cycles plus one sized top-up (about 331 USDG in total, all of which flows to Senior as coupon), `S` sits just below 2^256.

**The freeze.** From then on, any matured Junior deposit makes `quoteDepositFromState`'s `mulDiv` overflow (`Panic(0x11)`). The call is unguarded, so the **whole** `settleLpEpoch` reverts, including the Senior redemption funding the same call performs. A matured Junior deposit cannot be cancelled while `P > 0`, and no owner function skips it.

**Measured in the PoC** (repo fixture, release pool config, production `HousePool` / `TrancheVault` / `CfdEngine` / `OrderRouter`):
- a Senior LP's ~30,000 USDG redemption from a 300,000 USDG Senior tranche stays unfunded for **54 consecutive hourly settlements**;
- every attempt reverts with the same overflow;
- it is funded at hour 55 only because the FX weekend oracle freeze defers entries. Junior principal was still 40.9 USDG at that point.

Controls:
- the same redemption and deposit without the attack are funded at the first hour;
- the inflated pool with no matured Junior deposit pending is also funded at once.

## Vulnerability Details

Code at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

### 1. The wipe gate exempts only exact zero

```solidity
// packages/perps/src/HousePool.sol:915-920
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
if (!juniorActivationDeferred) {
    juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
}
```

With `P = 2,500` atoms (0.0025 USDG) and a large `S`, the deposit is still activated.

### 2. Deposit pricing multiplies the supply by D/P

`TrancheVault.quoteDepositFromState` (`TrancheVault.sol:1398-1410`) computes `mulDiv(net, pricingSupply + VIRTUAL_SHARES, pricingAssets + assets + 1 - net)`. With fee 0 the denominator is `P + 1`, so a deposit `D` mints `D*(S+1000)/(P+1)` shares.

### 3. The coupon brings P back to dust at a time anyone can compute

The Senior coupon is paid from `juniorPrincipal` continuously and is capped at the Junior (`src/libraries/HousePoolWaterfallAccountingLib.sol:64-90`). At 300k Senior and 8% APR it drains 761 atoms/s, so each deposit `D ≈ c·3h` returns to a few atoms about 3 hours later. The attacker computes the settlement second off-chain from public state.

`settleLpEpoch(0, 0)` is permissionless while no position is open, as during the whole attack in the PoC. With open positions, the router route with a fresh mark is used.

### 4. The overflow is unguarded and reverts the whole settlement

```solidity
// packages/perps/src/HousePool.sol:1046 (_settleDepositPhase)
uint256 cumulativeShares = vault.quoteDepositFromState(
    phase.acceptedAssets + epochAssets, phase.pricingAssets, phase.pricingSupply, feeBps
);
```

There is no try/catch, skip or deferral. Once `S` is close to 2^256, `mulDiv(net, S + 1000, P + 1)` overflows for any `net > P + 1`. OpenZeppelin `Math.mulDiv` reverts with `Panic(0x11)`, and the Senior redemption funding of the same call is rolled back with it.

### 5. The poisoned head cannot be removed

`TrancheVault.cancelPendingDeposit` (`:851-859`) refuses a matured Junior request with `DepositEpochAlreadyActive` unless `_projectedTerminalWipe()` holds. That requires `pricingAssets == 0` (`:1643-1647`), which is false while `P > 0`. `queueState.rejected` is set only when a quote yields 0 shares (`:1461-1465`), never on overflow. Anyone (the attacker included) can plant a Junior deposit and keep every settlement reverting.

The freeze ends only when:
- coupon drains the Junior to exactly 0; the attacker controls when, through the size of the final top-up, which becomes Junior principal;
- or an oracle-frozen window (FX weekend) defers entries before the Junior phase (`HousePool.sol:908-912`).

### Relation to prior audits and known issues

- **Not known.** Cecuro LOW-72 ("impossible share quotes for a wiped tranche", Fixed) concerns the view-only quote at exactly `P == 0`. This report shows that the exact-zero gate leaves the near-zero region open, which the program accepts as an incomplete fix. LOW-27, MED-19 and MED-24 concern zero-share fee accrual and bounded settlement work. The SECURITY.md Known Limitations (`:1328-1450`) say nothing about Junior share inflation, quote overflow or near-wipe pricing.
- **Contradicts the spec.** `ACCOUNTING_SPEC.md:326`: "a wiped tranche cannot be silently revived by an ordinary asynchronous deposit". A Junior at 2,500 atoms against 3.3e17+ shares is economically wiped, but the gate revives it on every deposit.

## Impact Details

**High, "Temporary freezing of funds for at least 24 hours".**

| Run (same fixture) | Hourly settlements reverted | Senior redemption funded at |
|---|---|---|
| Attack: inflated supply plus one 100 USDG Junior deposit | 54 (all `Panic(0x11)`) | hour 55 (only because of the weekend oracle freeze) |
| Control: no attack, same LPs, redemption and deposit | 0 | hour 1 |
| Control: inflated supply, no Junior deposit pending | 0 | hour 1 |

- **What is frozen:** every Senior LP redemption request, plus every Senior and Junior deposit activation, for as long as the poison is active. The PoC measures 54 hours for a 30k USDG redemption from a 300k USDG Senior tranche. The weekend window ended it while Junior principal was still 40.9 USDG.
- **Lasting damage:** Junior deposits of normal size stay dead while the supply sits near 2^256. This persists after the weekend.
- **Attacker cost:** about 331 USDG, which flows to Senior as coupon. That freezes the exits of 300k USDG of Senior capital (30k requested in the PoC). There is no attacker profit, so the claim is a freeze, not theft.

### Preconditions

- **Junior principal in the near-wipe region.** First loss is the Junior's designed role: in the PoC, one trader's profitable close leaves 6.35 USDG of Junior principal (from 100k), and the coupon drains the rest. Any decline of the Junior towards wipe passes through these values. This is a tail regime, but it is organic.
- Release pool parameters: 1 USDG seeds, 80% max Senior share, 8% Senior coupon.
- **The attacker settles at the second they choose.** `settleLpEpoch` is permissionless, and the timing is computed from public state. If another keeper settles first in an epoch, the attacker sizes the next deposit for that keeper's settlement time instead, because the drain is deterministic.
- **No role is needed.** The attacker only uses `requestDeposit` and `settleLpEpoch`.

## Attack scenario

1. Every cycle, the attacker:
   - requests a Junior deposit `D ≈ c·3h` (about 8 USDG);
   - calls `settleLpEpoch` at the second when projected Junior principal is about `3c` atoms, so supply grows about 3,000x;
   - waits about 3 h for the coupon to drain `D`.
2. After 17 cycles, supply has gone from 3.3e17 to 1.8e76.
3. A final top-up of about 190 USDG puts supply at 1.137e77, with headroom 2.07e75 below 2^256.
4. Any matured Junior deposit now overflows the quote; the PoC uses a stranger's 100 USDG. Every `settleLpEpoch` reverts, and Senior redemptions stay unfunded.

## Recommendation

Treat a Junior whose share price has collapsed below a dust floor as wiped for activation, not only `P == 0`. Apply the same predicate in `_projectedTerminalWipe()` so that pending Junior deposits stay cancellable:

```diff
-        bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
+        bool juniorActivationDeferred = juniorDepositPhase.pricingSupply != 0
+            && (juniorDepositPhase.pricingAssets == 0
+                || juniorDepositPhase.pricingSupply / juniorDepositPhase.pricingAssets > MAX_JUNIOR_SHARES_PER_ASSET);
```

The full diff is in the gist (`fix.diff`, `HousePool.sol` and `TrancheVault.sol`). With it:
- inflation stops after the third cycle, because the Junior activation is deferred;
- the attack test fails;
- the no-attack control passes;
- the repo's LP suites (`test/perps/spec/lp` and `accounting/waterfall`) pass 353/353.

As defense in depth, `_settleDepositPhase` should defer an epoch whose quote would overflow instead of reverting. Senior redemption funding should never depend on a Junior deposit quote succeeding.


## Link to Proof of Concept

https://gist.github.com/the-red-ace/f564f8439285756a32dde2002c40c5d0

## Proof of Concept

## Proof of Concept

**Gist:** https://gist.github.com/the-red-ace/f564f8439285756a32dde2002c40c5d0. It contains the test, two fixture files, `fix.diff` and a README.

**How the tests run:** in the repo's own Foundry suite at a82d4fe, on `BasePerpTest`.
- Fixtures: `X2GridBase` (release risk parameters and order helpers) and `JuniorWipeBase` (release pool config, LP helpers through public vault calls only).
- No in-scope contract is modified.
- The regime is built organically, through a trader profit, coupon accrual, public deposits and public settlements.
- Cheats: `vm.warp` and `vm.snapshotState` (to land the settlement second that the attacker computes off-chain), `vm.prank`, USDG mints to actors, and `MockPyth` prices. No `vm.store` or `deal` into protocol state.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/juniordust
cp X2GridBase.sol X2ErrNames.sol JuniorWipeBase.sol JuniorDustInflationFreeze.t.sol packages/perps/test/perps/juniordust/
forge test --root packages/perps --match-contract JuniorDustInflationFreezeTest -vv
```

| Test | What it shows |
|---|---|
| `test_SeniorRedemptionFrozenOver24h` | 17 cycles plus a top-up for 331 USDG. A ~30k Senior redemption then gets 54 hourly reverts, all `Panic(0x11)`, and is funded at hour 55. |
| `test_Control_NoAttack_RedemptionFundedFirstHour` | Same LPs, redemption and 100 USDG Junior deposit without the attack: funded at hour 1, no reverts. |
| `test_Control_InflatedButNoJuniorDeposit_RedemptionFunded` | Inflated supply, no Junior deposit pending: funded at hour 1. The pending Junior quote is what reverts the settlement. |

Expected output (reproduced on a clean a82d4fe checkout with only the gist files):
```
[PASS] test_Control_InflatedButNoJuniorDeposit_RedemptionFunded()
[PASS] test_Control_NoAttack_RedemptionFundedFirstHour()
[PASS] test_SeniorRedemptionFrozenOver24h()
  attacker USDG deposited in total (6dp): 331353838
  hourly settle attempts reverted: 54
  hours until the Senior redemption was funded: 55
  revert data: 0x4e487b710000000000000000000000000000000000000000000000000000000000000011
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

With `fix.diff` applied (`git apply fix.diff`), `test_SeniorRedemptionFrozenOver24h` fails. Inflation stops after the third cycle, and the attacker's next settlement has nothing to process (`HousePool__NoLpEpochProgress`). The no-attack control still passes.
