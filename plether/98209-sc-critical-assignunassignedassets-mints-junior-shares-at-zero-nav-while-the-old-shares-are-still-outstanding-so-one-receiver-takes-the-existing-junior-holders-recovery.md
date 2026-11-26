# \#98209 [SC-Critical] assignUnassignedAssets mints Junior shares at zero NAV while the old shares are still outstanding, so one receiver takes the existing Junior holders' recovery

**Submitted on Nov 5th 2026 at 23:57:54 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98209
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary
When Junior principal is 0 but Junior shares are still outstanding, the protocol treats the tranche as closed to new money:
- `requestDeposit` reverts `TerminallyWiped`;
- deposit activation is deferred;
- since the LOW-72 fix, `estimateDepositShares` returns 0.

`assignUnassignedAssets(false, receiver)` checks none of this. It mints `quoteBootstrapDeposit(U)` = `U × (supply + 1000) / (0 + 1)`. In the PoC, assigning 100,000 USDG against 1.0007e15 Junior shares mints 1.0007e26 new shares, which is 99.999999999% of Junior.

If the zero is a mark (an open trader gain), the old holders' USDG never left the pool. When the mark reverses, the recovery goes to the receiver:
- the existing holder ends with 12 atoms instead of 1,107,957.52 USDG (control);
- the receiver redeems 301,969.56 USDG with a quarter of its shares, three times the 100,000 it was assigned, and keeps about 906,000 more.

The amount assigned doesn't matter. An owner assigning just 1 USDG to itself gets 99.9999% of Junior, and a quarter of those shares redeems 276,959.73 USDG.

## Root cause
Deposits refuse exactly this state:

```solidity
// HousePool.sol:496, _canAcceptTrancheDeposits
: ctx.pendingState.waterfall.juniorPrincipal == 0 && ctx.pendingState.juniorSupply != 0) {
    return false;
}
// HousePool.sol:917, deposit activation
bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
```

The assignment checks only Senior capacity, then mints the quote. The quote has no guard (LOW-72 added one only to the estimate):

```solidity
// HousePool.sol:677-685, assignUnassignedAssets
if (toSenior) {
    _requireSeniorDepositCapacity(amount, _getWaterfallState());
}
...
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);

// TrancheVault.sol:1366, estimateDepositShares (LOW-72 fix)
if (pricingAssets == 0 && pricingSupply != 0) {
    return 0;
}
// TrancheVault.sol:1392-1396
function quoteBootstrapDeposit(uint256 assets) external view returns (uint256) {
    return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
}
```

## Why this is in scope
The program counts "a legitimate, documented action by a trusted role that triggers a bug harming users". Assigning ownerless pool assets is a listed owner power, and here the owner has no harmless option:
- **Entry stays blocked until U is assigned.** While `unassignedAssets > 0`, every deposit request and activation is blocked (ACCOUNTING_SPEC.md:324). This assignment is the only thing that lowers U (HousePool.sol:701). In the PoC, a Senior deposit accepted before the loss stays pending and uncancellable through 7 daily settlements. Only an LP settlement hold releases it, and the hold stops all LP settlement.
- **Senior is closed and Junior can't be pro rata.** With Junior at 0, Senior capacity is 0, so the Senior assignment reverts `ExceedsSeniorDepositCapacity`. The Junior assignment has a single receiver.
- **The owner can't size a recapitalization to avoid U.** `recapitalizePool` only queues the amount; the next settlement applies it at that settlement's mark (ACCOUNTING_SPEC.md:639-643), after routing new value through the waterfall (HousePool.sol:1568 before :1598). After a recap of exactly the previewed gap:
  - a routine 100 USDG margin top-up leaves U = 5.78;
  - one 10,000-notional trade leaves U = 10.27;
  - a 2% LP-favourable tick leaves U = 51,800.
- **No doc says to wait,** and waiting only helps if the zero later reverses. SECURITY.md:244-245 calls this "the guarded bootstrap path; these powers do not permit withdrawal of existing claimant backing". SECURITY.md:255-256 says the owner cannot "directly seize arbitrary user funds".

No malicious choice is needed. An owner assigning its own recapitalization surplus back to itself or the treasury gets the same result.

U > 0 with Junior at 0 arises in three ways:
- **(a)** a recap above the Senior gap (recapitalization "never accrues to junior principal", HousePoolPendingPreviewLib.sol:103-107);
- **(b)** a recap of exactly the gap, then any LP-favourable tick before the next settlement (the ordering in my report #98079);
- **(c)** any recap while Senior is at its high-water mark, e.g. one sized to restore E ≥ L before `clearDegradedMode` (SECURITY.md:246).

## Preconditions
- **The owner assigns** while Junior principal is 0, or small next to U.
- **Junior is marked to zero.** Every loss at or above Junior capital lands exactly on 0, so zero is not a knife edge. From a fresh structure it is a tail event, though: at release params (80/20, one-sided book at the 40% skew wall) the basket must rise 50.2%.
- **The realistic route is a depleted Junior.** The whitepaper's realized-cash replay (§8.6) reaches Junior principal $0 in all three scenarios, "partly because it funds the senior coupon". The PoC `AssignCaptureRealisticRouteTest` reproduces this at release params:
  - 80/20 at the Senior cap, the default 8% coupon and monthly settlements with no trading revenue drain Junior from 1,000,000 to exactly 0 in 33 months (1,040 days), with the shares still outstanding;
  - a short at 10% of pool cash and a +2% move put a 9,746 loss on Senior;
  - the owner recapitalizes exactly that gap, one small trade leaves U = 10.27, and the Junior assignment gives the receiver 99.99999%;
  - after the reversal the receiver holds 10,192 that the old holders keep in the control.

  That amount is small. It is mostly the owner's own recap reaching Junior, and it isn't withdrawable: Junior withdrawal capacity is 0 under the 80% covenant. The coupon passes it to Senior within about 9 days (the difference is 180.56 at +4 weeks). On this route the harm is the ownership transfer of a near-empty tranche. The large figures need the stress state.
- **Realized wipe:** nothing reverses, but future Junior revenue moves to the receiver. The old holder gets 9,122.06 of later revenue in the control and 1 atom after an early assignment.

Zero is the line because the price there no longer depends on the amount (1 USDG buys 99.9999%). Every public entry is closed, and the owner's mint is the only way in. The vendor test `test_OneDollarDepositCannotRecapWipedTranche` (TrancheRecoveryBehavior.t.sol:380) states the rule this path skips.

## Docs and known issues
- **ACCOUNTING_SPEC.md:326-327:** "a wiped tranche cannot be silently revived by an ordinary asynchronous deposit" and "seeded ownership continuity is preferred over governance re-assignment".
- **ACCOUNTING_SPEC.md:650:** "only value with no safe claimant path belongs in `unassignedAssets`". Here the surviving holders are a live claimant, and the assignment replaces them with one new owner.
- **Fee shares** minted at zero NAV share in a recovery by design (ACCOUNTING_SPEC.md:398-401). But they are pro rata and bounded by the fee rate (0 at release), so they never take the tranche.
- **LOW-72** called this quote "impossible" for a wiped tranche and zeroed only `estimateDepositShares`. `quoteBootstrapDeposit` computes the same quote and mints it. If this is judged the same root cause, that fix is incomplete.
- **MED-12** notes that the owner picks U's receiver. This report is about the price, which hands over the existing tranche.
- **Other audit items:** LOW-65, MED-26 and MED-5 are about seeding and activation. Nothing in Known Limitations or the AI scan covers this.
- **#98079** is about how U arises. The root cause and fix here are different: path (a) uses no same-pass revenue, and fixing #98079's ordering leaves this assignment unchanged.

## Suggested fix
- If the target tranche has shares outstanding, credit `amount` to its principal without minting, so the surviving holders keep it pro rata. Mint bootstrap shares only into a tranche with zero supply.
- At minimum, apply the deposit gate (`principal == 0 && supply != 0`) to the assignment, and add a non-minting release so U isn't stuck.



## Link to Proof of Concept

https://gist.github.com/chj2934/82a9b336eebea9fbefe17d8d94e93a2d

## Proof of Concept

## Proof of concept
PoC (secret gist): https://gist.github.com/chj2934/82a9b336eebea9fbefe17d8d94e93a2d

Save it as `packages/perps/test/poc/h-as-1/AssignCaptureZeroNav.t.sol` and run from `packages/perps`:

```
FOUNDRY_FFI=false forge test --match-path test/poc/h-as-1/AssignCaptureZeroNav.t.sol -vv
```

Both test contracts use:
- release risk params, seeds and Senior caps;
- public LP, trader and keeper calls through the Router;
- owner actions limited to `recapitalizePool`, `assignUnassignedAssets` and `pauseLpEpochSettlement`;
- no `pool.reconcile()` and no storage writes.

`AssignCaptureZeroNavTest` reaches the marked wipe the same way as #98079 (a 37%-of-depth short and a +75% move). `AssignCaptureRealisticRouteTest` uses 80/20, the coupon drain and a +2% move.

Result: 8 passed.

| Test | Result |
|---|---|
| `test_GuardsRefuseTheStateAssignDoesNot` | Estimate 0, deposit `TerminallyWiped`, Senior assignment reverts on capacity, Junior assignment mints 1.0007e26 shares against 1.0007e15 |
| `test_AssignAtZeroNav_ReceiverRedeemsOldHoldersRecovery` | Control: old holder 1,107,957.52. Capture: old holder 12 atoms; receiver redeems 301,969.56 |
| `test_ExactGapRecapPlusTick_SameCapture` | U 51,800; old holder 21 atoms; receiver redeems 276,953.29 |
| `test_NoHarmlessOption` | Senior deposit held 7 days; cancellable only under the hold; U stays |
| `test_OwnerSelfAssignOneUsdg` | Owner redeems 276,959.73 for a 1 USDG assignment |
| `test_P2_ExactGapRecap_UArisesBeforeNextSettlement` | Exact recap: U 0 if nothing happens; 5.78 once a margin top-up collects carry; 10.27 after one 10,000 trade |
| `test_P1_RealisticRoute_CaptureAndAftermath` | Junior drained to 0 in 1,040 days; receiver 99.99999%; old holder loss 10,191.98 after the reversal, 180.56 at +4 weeks |
| `test_P1_CapturedValueRedeemable` | Junior withdrawal capacity 0; the receiver's full redemption is funded 0 |

