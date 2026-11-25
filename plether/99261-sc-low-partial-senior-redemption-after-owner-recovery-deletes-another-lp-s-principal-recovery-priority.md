# \#99261 [SC-Low] Partial Senior redemption after owner recovery deletes another LP's principal-recovery priority

**Submitted on Nov 12th 2026 at 18:05:29 UTC by @Ailenia for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99261
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# 2. Description

## Brief

I found that a Senior LP can redeem only part of the live Senior share supply, receive the last one-atom Senior principal quote, and erase the full high-water mark (HWM) owed to the shares that remain. A later trader loss then flows to Junior instead of restoring the other Senior LP's impaired principal. My public-call PoC reaches this after an actual Senior impairment and an owner-funded recovery with surplus. In the matched control, the victim receives 51.967137 USDG; after my partial Senior exit, it receives zero. My complete controlled cohort gains 82.299245 USDG relative to the control and 120.999792 USDG over its supplied capital.

## Root cause

`TrancheVault._convertToAssetsUsingAssets` prices redemption with one virtual asset and 1,000 virtual shares. When Senior principal is one USDG atom, a holder with only part of the Senior shares can quote that full atom. `HousePool._settleLpEpoch` passes the funded atom to `HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw`, which computes `H' = H - floor(H * fundedAssets / seniorPrincipal)`. Because both asset amounts equal one, it deletes the whole HWM without checking that another Senior holder still owns shares. Later revenue has no Senior restoration target and is credited to Junior.

Cecuro MED-11 led to the funded-asset HWM formula, replacing a burned-share fraction. My PoC reaches the terminal boundary that fix missed: funding all remaining principal does not mean all protected shares exited. The published less-than-one-atom rounding tolerance concerns extra retained protection, while this path deletes a material HWM.

## Reproduction and prerequisites

My fixture seeds both tranches with 1 USDG and deposits 650 USDG into my Junior vault, 60 USDG into my Senior vault, 2,485 USDG into a second Senior address I control, and 55 USDG into the victim's Senior vault. A 1 USDG Junior dust holder supplies a one-share settlement head. After 3 years and 60 days of ordinary Senior coupon time, a public LP settlement drains Junior principal to zero. I then open a public 1,000-unit LONG at a 1.09 basket mark and close it one day later at 1.08. Public reconciliation records a real 9.482832 USDG Senior impairment.

The owner then calls the documented `recapitalizePool(60,000 USDG)` recovery action. It restores the Senior gap and places the remaining 59,990.517168 USDG in `unassignedAssets`, as the source's surplus-recapitalization branch specifies. The owner is separate from my controlled cohort; I cannot force its payment. The first-party `EngineRecapitalization` test also recovers an impaired Senior tranche with a surplus recap and checks that the surplus becomes unassigned. A second first-party waterfall invariant keeps a recap surplus reserved through later trading revenue. My second Senior address publicly redeems all its shares after the recap; its pending share count is zero, and 145.083666 USDG of protected Senior principal remains with my 60 USDG position and the victim's 55 USDG position. All later trading, settlement, redemption, claim, and refund calls are public.

I open a 20,000-unit LONG at 1.08 and close it one day later at 1.07. Reconciliation wipes both tranche principals but leaves the 145.083666 USDG Senior HWM and 54,830,291,705 victim Senior share units live. A small public LONG opened at 1.07 and closed 20 seconds later at 1.0699 transfers exactly one USDG atom of realized carry to pending Pool revenue. Public LP settlement assigns it to Senior principal while preserving the HWM. From this same state, my partial Senior redemption is funded for the atom and erases the HWM; the no-exit control settles in the same epoch and keeps the HWM.

A later public 20,000-unit LONG loses value from 1.07 to 1.075 over one day, supplying restoration revenue. The control victim publicly claims 51.967137 USDG. In the exploit, its request has a zero quote, pays zero, and refunds its shares; my Junior address claims 138.777672 USDG externally. My controlled cohort includes both Senior addresses, Junior, all trader addresses, the dust holder and sink, wallet balances, clearinghouse balances, trader claims, residual vault shares, and dust entitlements. I supplied 17,796 USDG to those roles. Their final value is 17,834.700547 USDG in the control and 17,916.999792 USDG in the exploit. The owner-funded 60,000 USDG is not counted as my capital or profit.

The same fixed prefunding passes with the target close at 1.071 and later mark at 1.076. It also passes with hourly mark updates through each market leg. A further 1.0699 later mark, without a material recovery rebound, still produces 3.601620 USDG of victim control cash versus zero after the HWM deletion. The PoC uses the released six basket feeds, weights, base prices, inversion flags, 2,606 raw confidence atoms per feed, 2,500-bps adverse-confidence multiplier, and released risk settings. MockPyth supplies structurally accepted external observations, not production-signed price history. Including the owner recap, trader prefunding, claimant deposits, and two seed assets, the fixture supplies approximately 77,853 USDG, below the competition's stated approximately 79,965 USDG TVL.

## Impact and remediation

Selected impact: **Direct theft of any user funds (at-rest or in-motion, other than unclaimed yield)**.

The victim's 51.967137 USDG control payout is below its original 55 USDG deposit, so the diverted value is recovery of paid-in principal, not unclaimed coupon or yield. My Junior claim is paid externally; a later owner assignment of the remaining unassigned assets can compensate a victim with new shares but cannot claw back that paid claim. My PoC executes such an assignment after the exploit and checks that my external Junior cash remains unchanged.

Preserve HWM protection for Senior shares that survive a partial redemption even when asset rounding lets that redemption consume the last principal atom. Pass the funded and remaining share counts into the HWM update, or track protected entitlement per share. Add a regression with `fundedAssets == seniorPrincipal == 1` atom, material HWM, and two Senior owners; later revenue must restore the surviving owner's principal before reaching Junior.

## References

- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:119-160`
- `packages/perps/src/HousePool.sol:624-701,824-861`
- `packages/perps/src/TrancheVault.sol:1861-1866`
- `packages/perps/src/CfdEngine.sol:627-651`
- `packages/perps/ACCOUNTING_SPEC.md:330-340,580-583,634-649`
- `packages/perps/SECURITY.md:155-163,1417-1419,1496-1505`
- `packages/perps/test/perps/spec/admin/EngineRecapitalization.t.sol:32-67`
- `packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol:307-319`
- Cecuro AI audit, MED-11, pages 30-31

## Link to Proof of Concept

https://gist.github.com/niafreu/1ae82b0e5f6e9253167a36aea580e1ed

## Proof of Concept

# 3. Proof of Concept

Secret gist: https://gist.github.com/niafreu/1ae82b0e5f6e9253167a36aea580e1ed

I ran the gist's test against the frozen `a82d4fefb02768e5b3bddcc97dee371eee24fea8` source. The PoC uses production Router, Engine, Pool, Vault, and Oracle logic. MockPyth supplies external price updates, and MockUSDG supplies test funds. Copy the gist file to `packages/perps/test/perps/poc/SeniorHwmPostMarkCarry60kPoC.t.sol`, then run:

```bash
cd packages/perps
FOUNDRY_PROFILE=quick forge test --match-path test/perps/poc/SeniorHwmPostMarkCarry60kPoC.t.sol --match-test test_GenuineRecoverySurplusHwmDeletesVictim -vv
```

Set `POC_HOURLY_MARKS=true` to replay hourly mark updates, or set `POC_P2=107100000 POC_P3=107600000` to test a second target interval point. Set `POC_P3=106990000` for the carry-driven arm without a material rebound. The test asserts the genuine pre-recap Senior deficit, surplus routing, completed filler exit, one-atom carry producer, normal mode, live victim shares, HWM erasure, same-state control, actual victim cash/refund, external Junior claim, complete controlled-cohort gain, and failure of later owner assignment to claw back paid Junior cash.
