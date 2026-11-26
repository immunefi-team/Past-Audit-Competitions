# \#99044 [SC-Critical] LP NAV drops the open VPI rebate but ignores its fully collateralised clawback, so a trader deposits Junior at the depressed price and takes the clawback from incumbent LPs (not fixed by the #97729 fix)

**Submitted on Nov 11th 2026 at 16:28:24 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99044
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

When an open reduces skew, HousePool pays the trader's VPI rebate R at once (CfdEngineSettlementSidecar.sol:442-443), and the same amount R of the trader's own funds is locked in a non-withdrawable reserve (:466-477; SECURITY.md:866-867). At close, the lifetime clamp charges R back and the reserve pays it into the pool (CloseAccountingLib.sol:112-118; CfdEngineSettlementSidecar.sol:691-693). Between open and close the pool therefore holds a receivable of exactly R, backed 1:1 by locked collateral.

Junior pricing ignores that receivable (SECURITY.md:871-872): deposits activate at the reconciled `juniorPrincipal` (HousePool.sol:915), which has already paid R. A trader who also deposits can:
1. take the rebate;
2. activate a Junior deposit at a price that is R too low;
3. close, which returns R to a pool it now largely owns;
4. redeem.

Incumbent Junior LPs pay. In the PoC (release parameters, mark held flat at 1.00, public routes), the attacker ends **+1,397.70 USDG** in absolute terms after all fees. The honest Junior LP (1M) ends 2,620.01 USDG below the control and 2,110.16 below its own starting value.

## A different root cause from #97729, and #97729's fix leaves it open

#97729 (my report, confirmed) is about a positive open charge that is booked as LP revenue but is refundable to the same position. Its recommended fix is to treat a live position's positive `vpiAccrued` as a pool liability. This report concerns the other item NAV omits: the clawback receivable on a negative `vpiAccrued`.

| | #97729 | This report |
|---|---|---|
| Item missing from LP NAV | Refundable positive VPI: an unfunded, contingent liability | Clawback of negative VPI: a receivable 100% collateralised by a locked reserve |
| NAV error | Too high | Too low |
| Attacker's LP action | Redeem before the refund | Deposit before the clawback |
| Who pays | LPs who stay | LPs who were already in |
| Code path | Revenue booking, CfdEngineSettlementSidecar.sol:454-464 | Rebate payout :442-443, reserve lock :466-477, clawback :691-693 |
| Fix | Reserve positive `vpiAccrued` | Count `max(-vpiAccrued, 0)` as a receivable |

`test_SeparateRoot_FullSkew` measures the separation:
- **The attacker never holds a refundable charge.** Its lifetime VPI is −R from open to close.
- **#97729's flow plays no part.** The honest trader's positive `vpiAccrued` is identical at the attacker's entry pricing and exit pricing. The honest LP's loss is already **2,613.78 USDG before that charge is ever refunded** (2,620.01 after).
- **The #97729 fix does not stop it.** Both LP prices are rebuilt from chain state with the vault's own formulas, which reproduce the shares minted and the assets paid to 1 atom. Re-pricing them with the honest charge held out of Junior NAV, as #97729's fix would do, gives an attacker LP gain of **2,614.89** (2,614.31 today) and an attacker total of **+1,398.28**. The fix makes this attack slightly better for the attacker.

## Attack (all public calls; no keeper, owner or guardian role)
1. An honest trader holds a skew-increasing LONG, as perps users routinely do.
2. Before the epoch cutoff, the attacker calls `TrancheVault.requestDeposit` on the Junior vault.
3. 60 s before the epoch boundary, the attacker opens the opposite SHORT. The pool pays R at once, and R of the attacker's funds is locked as the clawback reserve.
4. At the boundary, `OrderRouter.settleLpEpoch(bytes[])` activates the deposit at the NAV that already paid R.
5. The attacker closes. The clamp charges back exactly R whatever the pool depth (`pool gained at close` ≥ R), and the reserve is consumed.
6. After the 1 h cooldown the attacker redeems at the restored NAV.

The trader leg nets only execution fees, and price exposure is about 60 s. One cycle takes about 2 hours and can repeat whenever honest skew exists.

## Impact (PoC: 4M pool = 1M honest Junior + 3M honest Senior, mark 1.00 throughout)

| Test | Attacker deposit | Honest skew / attacker offset | Attacker absolute | Attacker vs control | Honest Junior vs control |
|---|---|---|---|---|---|
| `test_RebateFirst_FullSkew` | 12M | 0.38 × depth / 100% | **+1,397.70** | +2,616.39 | −2,620.01 |
| `test_RebateFirst_EqualStake` | 1M (same as the honest LP) | 0.38 × depth / 58% | **+455.09** | +1,162.12 | −1,162.95 |
| `test_RebateFirst_ModerateSkew` | 12M | 0.20 × depth / 55% | **+186.28** | +538.55 | −539.17 |

- **The loss is principal, not foregone yield.** In the control (same trades and clock, no LP round trip) the honest LP gains. In the full-skew attack it ends 2,110.16 below its starting value. Its 20% redemption pays out 524.00 USDG less than in the control.
- **Capital needed is modest.** The trader leg costs 2 × 4 bps of the offset notional. With a partial offset the attack pays once the attacker's Junior share f satisfies f × vpiFactor × skew / depth > 8 bps, i.e. f > ~0.21 at the 0.38 skew cap. The equal-stake test (f ≈ 0.5) needs only 1M against a 1M incumbent.
- **Collateral backing:** `test_RebateLowersJuniorNavWhileClawbackIsCollateralised` shows Junior deposit-pricing NAV falling by R (2,887.14 vs R = 2,887.13) while `vpiRebateReserveAssets(attacker) == R`, and staying down until the close.

## Why this is not intended behaviour or a known issue
- **SECURITY.md:1443** ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"). That holds for the trader leg, which nets to zero. The extraction is on the LP leg: deposit pricing ignores a clawback the protocol has already collateralised.
- **SECURITY.md:871-872** states the accounting choice. It is the same statement cited when #97729 was first closed, and the project has since confirmed #97729 as Critical.
- **Not depth manipulation.** The AI-scan VPI items concern depth manipulation, which this attack doesn't need: the clamp fixes the clawback at R at any depth.
- **Not listed anywhere.** It is not in Known Limitations, the Cecuro or pre-audit reports, or the AI scan.

## Recommendation
Count the reserve-backed clawback `max(-vpiAccrued, 0)` as a pool receivable wherever Junior is priced (deposit activation and redemption funding), or defer activation and funding while unsettled VPI exists. Combined with the #97729 fix, the result is NAV priced with signed anticipated VPI.

## Link to Proof of Concept

https://gist.github.com/chj2934/3c202a4aa43b7da4f2d524b49102e077

## Proof of Concept

## Proof of concept

PoC (secret gist): https://gist.github.com/chj2934/3c202a4aa43b7da4f2d524b49102e077

Save as `packages/perps/test/poc/vpi-mirror/VpiRebateMirrorSeparateRoot.t.sol` and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/vpi-mirror forge test --dynamic-test-linking --match-path test/poc/vpi-mirror/VpiRebateMirrorSeparateRoot.t.sol -vv
```

`FOUNDRY_TEST` and `--dynamic-test-linking` only skip compiling the vendor test files, which takes hours under via_ir. A plain `forge test --match-path …` gives the same result after the full build.

Result: 5 passed.

| Test | Shows |
|---|---|
| `test_RebateFirst_FullSkew` | Attacker +1,397.70 absolute; honest Junior −2,620.01 vs control |
| `test_RebateFirst_EqualStake` | 1M deposit: attacker +455.09 absolute; honest Junior −1,162.95 vs control |
| `test_RebateFirst_ModerateSkew` | Attacker +186.28 absolute at a 0.20 × depth skew |
| `test_RebateLowersJuniorNavWhileClawbackIsCollateralised` | Junior NAV −R while the reserve holds R |
| `test_SeparateRoot_FullSkew` | Loss before any positive-VPI refund; prices rebuilt to 1 atom; +1,398.28 with the #97729 fix applied |

Setup uses the vendor `BasePerpTest` with release risk parameters and the release Senior capacity (40M / 80%). After `setUp`, every `vm.prank` is an actor: the attacker's two wallets, the honest trader, the honest LP, or an unrelated settler. Execution goes through public `commitOrder`/`executeOrder`, `updateMarkPrice(bytes[])` and `settleLpEpoch(bytes[])`. MockPyth writes stand in for signed Pyth payloads and always carry the unchanged 1.00 mark.

