# \#99451 [SC-High] Any reconcile between a recapitalization and new revenue hands Senior's recovery to Junior

**Submitted on Nov 13th 2026 at 17:50:41 UTC by @nomorecaffeine for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99451
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# 2. Description

After the owner recapitalizes a wiped pool with R between the terminal deficit D and 2D, the next LP settlement before new revenue cuts the Senior high-water mark from R to the pending recap residual, so later revenue owed to Senior goes to Junior or to unassigned assets. In the PoC at R = 1.95D, Senior holders end 6,759.50 USDG lower than when the revenue arrives first, with equal total pool value.

The first reconcile after `recapitalizePool(R)` settles the backed R - D as Senior principal, sets the mark to R and leaves D pending, as the sponsor's `HousePoolWaterfall.t.sol:201` asserts. The next reconcile caps the pending D against distributable value (`HousePool.sol:1833`), and the planner nets the capped amount out of that same value:

[HousePoolReconcilePlanLib.sol#L68-L71](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71)

```solidity
        if (pendingBucketAssets > 0) {
            snapshot.distributable =
                snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
        }
```

For R <= 2D that value is the R - D backing Senior, so both tranches mark to 0 and the capped recap takes the empty-pool bootstrap branch with the residual D as target:

[HousePoolPendingPreviewLib.sol#L116-L122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L116-L122)

```solidity
        uint256 remaining = amount;
        if (state.seniorSupply > 0) {
            if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
                state.waterfall.seniorPrincipal += remaining;
                state.waterfall.seniorHighWaterMark =
                    recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
                remaining = 0;
```

The mark becomes max(D, R - D) = D, not the R that `ACCOUNTING_SPEC.md:338` restores first. The zero marks are derived, not logged. The cap is the pre-audit SC-L1 fix, whose response promises a residual "stays pending, stays reserved everywhere, and gets applied later once it’s backed". At R = D nothing settles, and above 2D the marks stay positive.

Any reconcile before new revenue has this effect, including the owner's own documented senior-rate change: `finalizePoolConfig` reconciles when the rate changes (`HousePool.sol:336-341`), so with no stranger or redeem head it drops the mark at R = 1.25D from 34,289.83 to 27,431.86 and the pending recap from 27,431.86 to 20,573.90. After the same 60,029.37 of revenue Senior holds 27,431.86 instead of 34,289.83.

It becomes a capture with `_requireMinimumRedeemSize` (`TrancheVault.sol:1611-1618`), which exempts a full-balance request, so worthless Junior shares still make a redeem head. With the `DEPLOYMENT.md` risk parameters:

- Junior holds 100,000 USDG (Alice: 1% in address A, 50% in address B); Senior holds 110,000.
- The basket goes 1.00 -> 1.60 -> 0.40, wiping both tranches with D = 27,431.86 USDG, far outside the 22.67% largest historical decline (`WHITEPAPER.md:1392`).
- Following the runbook (`SECURITY.md:1509`), the owner recapitalizes R = 1.25D = 34,289.83 and clears degraded mode; a stranger's settlement sets the mark to 34,289.83 with 27,431.86 pending.
- Alice requests A's full balance and settles on that head:
  - the mark drops to 27,431.86 with no new cash.
- The pool earns 60,029.37 USDG from 40 losing trades.
- B redeems 9,440.69 USDG. Without Alice's request, B holds an unredeemable 2,582.80 mark (settles revert `HousePool__NoLpEpochProgress`) and receives 0.

Which head lands the settlement depends on R: at 1.25D a Senior redeem head reverts and a Junior head works, and at 1.95D an ordinary Senior redeem lands it with no attacker. The owner chooses R; at R = D the pool accepts no new trades.

**Impact.** Senior holders lose the restoration to their high-water mark that later revenue owes them, to Junior shares or unassigned assets. A Junior holder can trigger it with shares already worth 0 and redeem the diverted revenue. Nothing in `src` restores the mark; `assignUnassignedAssets` mints new shares to one receiver.

## Caller authorization

No role is used by the attacker: A's request goes through `TrancheVault.requestRedeem` and every settlement through the permissionless `OrderRouter.settleLpEpoch`, while the owner's `recapitalizePool` and `clearDegradedMode` are the documented terminal-deficit response, which the program places in scope ("In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol."). The program also states "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope."

## Recommendation

In `applyClaimantRecapitalizationIntent`, skip the empty-pool bootstrap for the residual of a recapitalization whose bootstrap already set the mark, and route it through the gap branch instead. Alternatively, have `HousePool._reconcile` stop netting that residual out of distributable value already backing Senior principal.

## Link to Proof of Concept

https://gist.github.com/nguyen-xuan-quoc/2cc1852938d8b4ebb18ff90784d99ddc

## Proof of Concept

# 3. Proof of Concept

https://gist.github.com/nguyen-xuan-quoc/2cc1852938d8b4ebb18ff90784d99ddc

The PoC runs in the repository's Foundry test suite at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, without modifying any contract in scope.

1. Copy `01-RecapResidualHwmCollapse.t.sol` from the Gist to `packages/perps/test/perps/poc/RecapResidualHwmCollapse.t.sol`.
2. From `packages/perps`, run:

```sh
forge test --match-path test/perps/poc/RecapResidualHwmCollapse.t.sol -vv
```

USDG and Pyth are mocks, orders go through the suite's `LegacyOrderRouterHarness`, which inherits the production `OrderRouter`, and every `settleLpEpoch` is sent by an address holding no role. The full log is `02-run-output.txt` in the Gist.

Controls: `test_SecondSettleBeforeRecoveryCutsSeniorRestoration` is the passive case at R = 1.95D (mark 53,492.13 to 24,938.10, D scaled for the redeemer's exit; Senior holders 34,066.67 vs 27,307.17 USDG), `test_OwnerRateChangeReconcileCollapsesSeniorHwm` is the owner rate-change case at R = 1.25D (control: finalize after the revenue), and `test_RecapEqualToDeficitClearsDegradedWithoutHwmCollapse` leaves the mark unchanged at R = D.

**Expected:** the residual is applied later toward the mark of R, and A's request changes nothing.

**Actual:** all four tests pass; the attack test logs:

```
[PASS] test_JuniorHolderStrikeTurnsSeniorRecoveryIntoRedeemableUsdg() (gas: 323182772)
Logs:
     first settle: Senior HWM / pending recap 34289825420 27431860336
  -- attack: A full-balance redeem is the strike head
     D / R 27431860336 34289825420
     strike: Senior HWM before / after 34289825420 27431860336
     B mark after recovery 9440693152
     B realized USDG 9440693152
     first settle: Senior HWM / pending recap 34289825420 27431860336
  -- control: holder does nothing
     D / R 27431860336 34289825420
     B mark after recovery 2582796540
     settleLpEpoch reverted, error data:
  0x86cca6b8
     settleLpEpoch reverted, error data:
  0x86cca6b8
     settleLpEpoch reverted, error data:
  0x86cca6b8
     settleLpEpoch reverted, error data:
  0x86cca6b8
     B realized USDG 0
  == R = 1.25D: B realized USDG, attack / control 9440693152 0
  == recovery revenue (both runs) 60029365327
```

