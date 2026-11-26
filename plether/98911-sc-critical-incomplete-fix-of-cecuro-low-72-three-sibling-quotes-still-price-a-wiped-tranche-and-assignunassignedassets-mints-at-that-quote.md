# \#98911 [SC-Critical] Incomplete fix of Cecuro LOW-72: three sibling quotes still price a wiped tranche, and assignUnassignedAssets mints at that quote

**Submitted on Nov 10th 2026 at 21:31:25 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98911
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
Cecuro LOW-72 ("Pending depositor can receive impossible share quotes for a wiped tranche", marked "Addressed") was fixed in commit `d757d0b1` by adding a wiped-tranche guard to one function, `TrancheVault.estimateDepositShares`:

```solidity
// TrancheVault.sol:1361-1370; lines 1364-1369 are the only change d757d0b1 makes to a quote function
// (in this and the next block, multi-line signatures are joined on one line)
function estimateDepositShares(uint256 assets) public view returns (uint256) {
    uint256 pricingAssets = _depositPricingAssets();
    uint256 pricingSupply = accruedTotalSupply();
    if (pricingAssets == 0 && pricingSupply != 0) {
        return 0;
    }
    return quoteDepositFromState(assets, pricingAssets, pricingSupply, 0);
}
```

Three sibling quotes of the same vault read the same pricing input and did not get the guard. After a terminal Junior wipe (Junior principal 0, share supply still outstanding) they keep quoting against zero pricing assets:

```solidity
// TrancheVault.sol:515-519
function convertToShares(uint256 assets) public view override returns (uint256) {
    return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
}
// TrancheVault.sol:1372-1376
function estimateMintAssets(uint256 shares) external view returns (uint256) {
    return _previewMintAssets(shares);   // :1872 -> _convertToAssetsUsingAssets(shares, _depositPricingAssets(), Ceil)
}
// TrancheVault.sol:1392-1396
function quoteBootstrapDeposit(uint256 assets) external view returns (uint256) {
    return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
}
// TrancheVault.sol:1848-1859, the shared helper pair (untouched by the fix)
function _depositPricingAssets() internal view returns (uint256 assets) {
    (uint256 seniorPrincipalAssets, uint256 juniorPrincipalAssets) = POOL.getPendingDepositTrancheState();
    return IS_SENIOR ? seniorPrincipalAssets : juniorPrincipalAssets;
}
function _convertToSharesUsingAssets(uint256 assets, uint256 totalAssets_, Math.Rounding rounding) internal view returns (uint256) {
    return Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1, rounding);
}
```

With pricing assets 0 the denominator is 1, so the share quote is `assets x (surviving supply + 1000)`, and the asset quote for the whole supply rounds to 1 atom.

The helper pair is the code LOW-72 itself pointed at. The audit item lists three locations: `TrancheVault.sol:1298-1308`, `:1344-1348` and `:1827-1838`. In the file as it stood before the fix (parent of `d757d0b1`) these are `maxRequestDeposit`, `estimateDepositShares`, and exactly `_depositPricingAssets` (`:1827-1830`) plus `_convertToSharesUsingAssets` (`:1832-1838`). The fix guarded one caller and left the helper pair and its three other callers as they were. `git blame` at the frozen commit: only `:1364-1369` come from `d757d0b1` (2026-10-22); `convertToShares` was last changed on 2026-05-18, `estimateMintAssets` and `quoteBootstrapDeposit` on 2026-08-21.

**Measured in the wiped state, before any owner action** (PoC test 1; Junior share supply 200,317,299,120,011; quote size 10,000 USDG, the size used in LOW-72):

| Quote | Healthy | Wiped, frozen code | Wiped, same guard on the siblings |
|---|---|---|---|
| `estimateDepositShares(10,000 USDG)` (fixed by LOW-72) | 10,016,241,826,614 | **0** | 0 |
| `convertToShares(10,000 USDG)` | 10,016,241,826,614 | **2,003,172,991,210,110,000,000,000** | 0 |
| `quoteBootstrapDeposit(10,000 USDG)` | 10,016,241,826,614 | **2,003,172,991,210,110,000,000,000** | 0 |
| `estimateMintAssets(whole Junior supply)` | 199,992,474,811 atoms (199,992.474811 USDG) | **1 atom** | 0 |

In one state the vault's paired estimates disagree: the deposit estimate says "0 shares", while the mint estimate says the whole Junior supply costs 0.000001 USDG and the two share quotes are about 2 x 10^11 times the healthy quote. These are view calls; they need no role.

The program admits this kind of report: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." (program page, Known Issues and Previous Audits).

**Not a documented exception.** The interface comment of `estimateMintAssets` says it "does not enforce the deposit-request minimum or other admission/settlement gates" (`IAsyncTrancheVault.sol:50-51`). The comment of the fixed function names the wipe rule separately from those gates: "Returns zero when projected deposit-pricing assets are zero with surviving shares. The request minimum and other admission/settlement gates are not encoded" (`IAsyncTrancheVault.sol:41-43`). So in the project's own wording the wipe rule is not one of the gates the mint estimate leaves out, and no comment or document states what `convertToShares`, `estimateMintAssets` or `quoteBootstrapDeposit` return for a wiped tranche.

## Vulnerability Details
**One of the three quotes is executed on-chain.** `HousePool.assignUnassignedAssets` prices its mint with `quoteBootstrapDeposit` (pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`):

```solidity
// HousePool.sol:685-688 - the quote, then the only check on it
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
if (shares == 0) {
    revert HousePool__BootstrapSharesZero();
}
// HousePool.sol:698-702 - principal is credited after the quote, then the quoted shares are minted
} else {
    juniorPrincipal += amount;
}
unassignedAssets = 0;
ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver);
```

- The quote reads Junior pricing assets before `amount` is credited (`:685` runs before `:699`), so in the wiped state it is `amount x (surviving supply + 1000)`.
- The zero check at `:686-688` is the place where a guarded quote would stop the call. With the unguarded quote it passes.
- `bootstrapMint` (`TrancheVault.sol:1549-1555`) mints exactly the quoted number to the receiver.

**How the state is reached (PoC test 2, release risk parameters, Senior cap 40M / 80 %).** Junior holds 200,000 USDG from one LP plus the 1 USDG seed; Senior holds 800,000 plus its seed. A trader's gain larger than Junior NAV wipes Junior principal to 0 and is realized (the trader closes) before any owner action. The owner recapitalizes with 100,000 USDG through `CfdEngine.recapitalizePool`. Reconciliation restores Senior to its high-water mark (798,816.211738) and books the remainder, 47,261.138536 USDG, as `unassignedAssets`. While that bucket is nonzero, deposits are blocked ("`unassignedAssets > 0` blocks new tranche-deposit requests and deposit activation", `ACCOUNTING_SPEC.md:324`). Assigning to Senior reverts `HousePool__ExceedsSeniorDepositCapacity` (measured), so the Junior target is the only call that clears the bucket.

**Measured result of the assignment to Junior** (amounts in USDG):

| | Before the assignment | After the assignment | After later Junior revenue |
|---|---|---|---|
| Receiver: shares | 0 | 9,467,223,624,915,451,899,379,896 = `quoteBootstrapDeposit(unassigned)` exactly | same |
| Receiver: share of Junior supply | 0 | 99.9999999978840966 % | same |
| Receiver: redeemable value | 0 | 47,261.138536 (the assigned amount) | 52,261.082700 |
| Pre-wipe Junior holders: redeemable value | 0 | 0 | 0.000001 |
| Senior principal | 798,816.211738 | 798,816.211738 | 798,424.095611 |

- At the assignment no value moves between parties: the receiver's value equals the assigned amount, the pre-wipe Junior holders are at 0 before and after, Senior is unchanged.
- The effect is on what comes later. The pre-wipe holders keep their shares, but those shares are now about 2.1 x 10^-11 of the supply. Later Junior revenue in the PoC (another trader's realized loss) raises Junior principal by 4,999.944165: the receiver's value grows by 4,999.944164 and the pre-wipe holders' by 0.000001.
- With the same guard on the siblings, the same owner call reverts `HousePool__BootstrapSharesZero` while Junior pricing assets are 0 (PoC tests 2 and 3).

## Impact Details
**Impact (Low):** "Contract fails to deliver promised returns, but doesn't lose value"

- **Doesn't lose value.** No principal of any user is lost through this. The pre-wipe holders' value is 0 before any owner action (their loss came from the trader's gain) and 0 right after the assignment; the receiver holds exactly the assigned amount; Senior is unchanged.
- **Fails to deliver.** (1) In the wiped state the vault returns share and asset quotes that no settlement performs, which is the defect LOW-72 described for "the vault's quote surface" and that the team answered by returning 0. Integrators reading `convertToShares` or `estimateMintAssets` still get them. (2) The owner's assignment is minted at such a quote, and afterwards the pre-wipe Junior shares take part in later Junior revenue at about 2.1 x 10^-11 of supply. The accounting spec says of zero-NAV Junior shares that they "participate in a later recovery like ordinary transferable Junior shares" (`ACCOUNTING_SPEC.md:400-401`; that sentence is written about maintenance-fee shares).
- **Why this is not filed higher.** The guard alone does not decide who receives the later Junior revenue. The PoC runs two controls with the guard installed on the siblings:
  - Assignment after the later revenue: the pre-wipe holders end with 5,007.239290 and the receiver with 47,261.138535.
  - Assignment at the first positive Junior NAV (one trade's fees put 7.239290 USDG into Junior, so the guard condition `pricingAssets == 0` is false): the receiver gets 99.9846847060703895 % of the supply and the pre-wipe holders end with 8.003938, against 0.000001 on the frozen code.

  So the part of the shift that the missing guard causes, measured against the earliest assignment the guarded code allows, is 8.003937 USDG in this scenario. In the second scenario of the PoC (test 3: the loss is unrealized at the assignment and the mark later returns to 1.00) it is 88.905185 USDG (pre-wipe holders 0.000006 on the frozen code, 88.905191 with the guard; receiver 300,230.655664 and 300,155.769621). The rest of the shift follows from when the owner assigns, which the guard does not control. We therefore map the finding to the Low row and leave the final mapping to triage.
- **Preconditions.** For the quotes: a terminal Junior wipe (Junior principal 0 with surviving supply), which the protocol documents and handles as a state (`ACCOUNTING_SPEC.md:326`). For the assignment: in addition an owner recapitalization larger than the Senior gap, and the assignment made while Junior pricing assets are exactly 0. The price move in the PoC (1.00 to 0.35) is only the way the test reaches the wiped state; any loss larger than Junior NAV does.

## Trusted role
The PoC contains owner calls, so this section states exactly what the owner does. It performs two actions, both listed for the role on the program page: "Owner of each perps contract (set by Plether at deployment): [...] Immediately, can pause and unpause, set the treasury, recapitalize the pool, clear degraded mode, seed the tranches and activate trading, admit or sweep unaccounted USDG, assign ownerless pool assets, and rescue tokens other than USDG." In the scenario the owner changes no parameter; the fixture only sets the release Senior cap during setup. The owner does not act "maliciously", is not "compromised", and does not act "outside its documented operational procedures" (the three cases the program page puts out of scope).

The project's documents describe both actions:
- Recapitalization: "`CfdEngine.recapitalizePool(amountAssets)` is an immediate, owner-only, non-reentrant recovery action." (`SECURITY.md:157`); "4. Recapitalize or clear positions." (`SECURITY.md:1509`, terminal-deficit runbook); "Normal reconciliation allocates the queued amount to senior restoration or unassigned assets" (`ACCOUNTING_SPEC.md:642`).
- Assignment: "assign unowned accounted assets through the guarded bootstrap path" (`SECURITY.md:244-245`, owner powers); "Explicitly bootstraps quarantined LP assets into a tranche by minting matching shares." (`HousePool.sol:653`); unassigned assets "Require explicit assignment; cannot silently become windfall LP ownership." (`INTERNAL_ARCHITECTURE_MAP.md:75`); the accounting spec names `assignUnassignedAssets` as one of the "bootstrap admin flows" (`ACCOUNTING_SPEC.md:320`).

The program rules for this case: "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol." (Trusted Roles and Non-Public Services) and "[...] where an in-scope impact arises from a logic error that a privileged address can trigger, which remains valid." (Attacks Involving Access to Privileged Addresses for Smart Contract Impacts).

Two objections, answered up front:
- *"The owner chooses the receiver."* True, and this report does not depend on that choice. Whichever address is named receives `amount x (surviving supply + 1000)` shares. The report is about the quote the contract computes for the mint, not about whom the owner picks.
- *"No document has a numbered 'assign after a wipe' step."* Also true: the assignment is documented as an owner power and as the required explicit path for unassigned value, not as a runbook step for this state. The first part of the report needs no owner at all: the three quotes return the values in the first table to any caller as soon as the tranche is wiped (PoC test 1 makes no owner call after the fixture setup). The assignment only shows where one of those quotes is executed.

## Relation to audit items
- **LOW-72 (Cecuro, "Addressed").** Same root cause: wiped-tranche pricing on the vault's quote surface. This report shows the fix at the frozen commit is incomplete: one of four quotes is guarded, and the helper pair named in the item's own locations is unchanged. LOW-72 names neither the three sibling functions nor the assignment.
- **LOW-22 (Cecuro, "Addressed").** A different claim. LOW-22 is about the missing non-dilutive recapitalization entry point and was fixed by adding `CfdEngine.recapitalizePool`, which the PoC uses as designed. It mentions "bootstrap assignment" only as an "economically different" alternative. We do not dispute LOW-22; it does not say what the vault quotes for a wiped tranche.

## Recommended fix
Apply the LOW-72 guard to the three siblings, or once where they share it (the deposit-side pricing built from `_depositPricingAssets()`):

```solidity
if (_depositPricingAssets() == 0 && accruedTotalSupply() != 0) {
    return 0;
}
```

The PoC control installs exactly this in `convertToShares`, `estimateMintAssets` and `quoteBootstrapDeposit` (one added statement in each of the three functions; the change is listed in the PoC section). With it, all four quotes return 0 in the wiped state, and `assignUnassignedAssets` into a wiped tranche reverts at the existing zero check (`HousePool.sol:686-688`) until Junior pricing assets are positive. If the team wants the assignment to stay available in the wiped state, it needs its own stated price rule for that case; that is a design decision for the team.

## References
- `packages/perps/src/TrancheVault.sol#L515-L519`, `#L1361-L1376`, `#L1392-L1396`, `#L1549-L1555`, `#L1848-L1873`
- `packages/perps/src/HousePool.sol#L653-L704` (`assignUnassignedAssets`), `#L1330-L1338` (`getPendingDepositTrancheState`)
- `packages/perps/src/interfaces/IAsyncTrancheVault.sol#L39-L54`
- Fix commit `d757d0b175b5c2154babee946362ff90bd585f22` ("fix(perps): resolve LP vault, tranche, and claim findings (#123)", 2026-10-22); Cecuro's fix link for LOW-72: plether-core `899675594d1e043383e10252a8985bac2e5b6271`
- Cecuro AI audit report (`audits/plether-ai-audit-report-2026-10-26.pdf`): items LOW-72 and LOW-22
- `packages/perps/ACCOUNTING_SPEC.md` l.320, l.324, l.326, l.400-401, l.642; `packages/perps/SECURITY.md` l.157, l.244-245, l.1509; `packages/perps/INTERNAL_ARCHITECTURE_MAP.md` l.75

## Proof of Concept

## Proof of Concept
One runnable Foundry test file in the repository's own perps test suite, at the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It is a local test; no fork, RPC or env var is used.
- **Fixture:** the repository's `BasePerpTest`. The only mocks are MockPyth and the USDG mock, both outside the scope (the program page allows mocks for "components outside the scope, such as Pyth price updates and USDG").
- **Parameters:** release values (MM 10 bps, IM 20 bps, frozen-market margin 300 bps, Senior cap 40M USDG and 80 % of the pool).
- **In-scope contracts are not modified** in any branch that shows the reported behaviour (test 1 up to its control line, and the "FROZEN CODE" branches of tests 2 and 3). These branches run first, before any cheatcode touches a protocol contract.
- **Control:** `LOW72GuardedTrancheVault.sol` is a copy of `src/TrancheVault.sol` with the LOW-72 guard added to `convertToShares`, `estimateMintAssets` and `quoteBootstrapDeposit`. The control branches install its code at the Junior vault address with `vm.etch` (storage and immutables unchanged). It is used only to show what the same steps do with the guard in place.
- No `vm.store`. No protocol address is impersonated.

### Setup and command
```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive

# 1. Save the test code at the end of this section as packages/perps/test/LOW72SiblingQuotes.t.sol

# 2. Create the control copy of the vault (the guard on the three siblings). Four lines change; nothing else.
G='if (_depositPricingAssets() == 0 \&\& accruedTotalSupply() != 0) return 0; '
sed -e 's|^pragma solidity 0.8.35;|& // TEST-ONLY copy of src/TrancheVault.sol: the wiped-tranche guard of estimateDepositShares added to three more quotes|' \
    -e 's/^contract TrancheVault is/contract LOW72GuardedTrancheVault is/' \
    -e "s/return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);/${G}&/" \
    -e "s/return _previewMintAssets(shares);/${G}&/" \
    packages/perps/src/TrancheVault.sol > packages/perps/test/LOW72GuardedTrancheVault.sol
git diff --no-index --stat packages/perps/src/TrancheVault.sol packages/perps/test/LOW72GuardedTrancheVault.sol   # 5 insertions, 5 deletions

# 3. Run
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/LOW72SiblingQuotes.t.sol -vv   # Foundry 1.5.1
```

Expected result: `3 passed; 0 failed`. `FOUNDRY_SPARSE_MODE=true` compiles only this test and its dependencies (195 files, about one minute); without it forge first builds the whole via-IR suite. The compiler prints warnings for `lib/forge-std`, OpenZeppelin `ERC4626.sol` and `src/CfdEngine.sol`; none is in the two PoC files.

The control copy differs from `src/TrancheVault.sol` in these lines only (the `git diff --no-index` in step 2 shows the same):
```diff
-pragma solidity 0.8.35;
+pragma solidity 0.8.35; // TEST-ONLY copy of src/TrancheVault.sol: the wiped-tranche guard of estimateDepositShares added to three more quotes
-contract TrancheVault is ERC4626, IAsyncTrancheVaultClaimableRedeem {
+contract LOW72GuardedTrancheVault is ERC4626, IAsyncTrancheVaultClaimableRedeem {
 // convertToShares (:518) and quoteBootstrapDeposit (:1395)
-        return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
+        if (_depositPricingAssets() == 0 && accruedTotalSupply() != 0) return 0; return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
 // estimateMintAssets (:1375)
-        return _previewMintAssets(shares);
+        if (_depositPricingAssets() == 0 && accruedTotalSupply() != 0) return 0; return _previewMintAssets(shares);
```

### Steps (amounts in USDG; the log prints 6-decimal atoms)
**Shared start.** Junior: one LP deposits 200,000, plus the 1 USDG seed. Senior: one LP deposits 800,000, plus the 1 USDG seed. Healthy quotes for 10,000 USDG: 10,016,241,826,614 shares from all three share quotes; `estimateMintAssets(whole supply)` = 199,992,474,811 atoms. TRADER_A opens LONG 3,900 lots at 1.00; the basket moves 1.00 to 0.35; settlement reconciles. Junior principal is 0, the Junior share supply (200,317,299,120,011) survives, and `maxRequestDeposit` is 0.

**Test 1, `test_1_QuoteTable_WipedJunior_NoOwnerAction`.** No owner call after the fixture setup; the quote functions are views that any caller can read.
- Unmodified contracts: `estimateDepositShares(10,000)` = 0; `convertToShares(10,000)` = `quoteBootstrapDeposit(10,000)` = 2,003,172,991,210,110,000,000,000 = `10,000e6 x (supply + 1000)` (asserted); `estimateMintAssets(whole supply)` = 1 atom.
- Control (guard on the siblings): all four return 0.

**Test 2, `test_2_AssignmentMintsAtTheUnguardedQuote_RealizedLoss`.**
1. TRADER_A closes at 0.35, so the Junior loss is realized before any owner action. Pre-wipe Junior holders: 0.
2. The owner recapitalizes with 100,000 (`CfdEngine.recapitalizePool`). After reconciliation Senior is back at its high-water mark, 798,816.211738, and 47,261.138536 is unassigned. The wiped-state quotes are the same as in test 1.
3. FROZEN CODE (unmodified contracts). The owner calls `assignUnassignedAssets(true, receiver)`: reverts `HousePool__ExceedsSeniorDepositCapacity`. The owner calls `assignUnassignedAssets(false, receiver)`: the receiver gets 9,467,223,624,915,451,899,379,896 shares, equal to `quoteBootstrapDeposit(unassigned)` read just before the call and to `unassigned x (supply + 1000)` (both asserted), which is 99.9999999978840966 % of the Junior supply. Receiver's value 47,261.138536; pre-wipe holders 0; Senior 798,816.211738.
4. Later Junior revenue: TRADER_B opens LONG 1,000 lots at 0.35, the basket moves to 0.40, TRADER_B closes. Junior principal grows by 4,999.944165; the receiver's value grows by 4,999.944164 to 52,261.082700; the pre-wipe holders' value grows by 0.000001.
5. GUARD-LATE (control, from the snapshot of step 2). All four quotes are 0; the Junior assignment reverts `HousePool__BootstrapSharesZero`. After the same TRADER_B round trip the pre-wipe holders hold 5,007.239290. The owner then assigns: receiver 47,261.138535, pre-wipe holders still 5,007.239290.
6. GUARD-EARLY (control, from the snapshot of step 2). The Junior assignment reverts; TRADER_B opens and its fees put 7.239290 into Junior pricing assets; the owner assigns now: the receiver gets 99.9846847060703895 % of the supply. After TRADER_B closes: pre-wipe holders 8.003938, receiver 52,253.082815. Difference for the pre-wipe holders between GUARD-EARLY and FROZEN CODE: 8.003937.

**Test 3, `test_3_Sensitivity_UnrealizedLossThenMarkReversal`.** TRADER_A stays open. The owner recapitalizes with 100,000 (47,273.241241 unassigned).
- FROZEN CODE: the owner assigns to Junior; the basket returns to 1.00 and TRADER_A closes. Receiver 300,230.655664; pre-wipe holders 0.000006.
- GUARD-EARLY (control): the assignment reverts; a fee-only TRADER_B round trip puts 14.002184 into Junior pricing assets; the owner assigns; same reversal. Receiver 300,155.769621; pre-wipe holders 88.905191. Difference for the pre-wipe holders: 88.905185.

### Every privileged or impersonated call, and the on-chain role it stands for
- **The test contract (`address(this)`) is the owner** of every perps contract: `BasePerpTest` deploys them, and they are `Ownable` to the deployer. Owner calls need no prank:
  - in the fixture setup: `pool.initializeSeedPosition` (Junior and Senior seed, 1 USDG each, held by the test contract), `pool.activateTrading`, and `pool.proposePoolConfig` + `pool.finalizePoolConfig` after the 72-hour delay, which set the release Senior cap (40M USDG, 80 %);
  - `engine.recapitalizePool(100,000e6)`: the owner's "recapitalize the pool";
  - `pool.assignUnassignedAssets(true | false, RECEIVER)`: the owner's "assign ownerless pool assets".
- **`vm.prank(KEEPER)`** (0x0E0E1) = any caller acting as keeper, which the program page defines as untrusted: `router.updateMarkPrice`, `router.settleLpEpoch`, `pool.settleLpEpoch(0, 0)`, `router.executeOrder`.
- **`vm.prank(TRADER_A / TRADER_B)`** = ordinary traders: `router.commitOrder`; in the fixture's `_fundTrader`, `approve` and `clearinghouse.deposit` from their own wallets.
- **`vm.prank / vm.startPrank(JUNIOR_LP / SENIOR_LP)`** = ordinary LPs acting on their own positions: `approve`, `requestDeposit`, `claimDeposit`, `requestRedeem`. The Senior LP's small redemption requests (1/2000 of its shares) are only a permissionless way to make settlement reconcile; they run on the same schedule in every branch.
- **RECEIVER** (0xB007) = the account the owner names. It makes no call.
- No protocol address (pool, engine, vault, router) is impersonated. `vm.store` is not used.
- Test scaffolding: `settlementToken.mint` (USDG mock) funds the accounts and the owner; MockPyth supplies prices; `vm.warp` / `vm.roll` move time; `vm.snapshotState` / `vm.revertToState` start the branches of one test from the same state.
- `vm.etch(juniorVault, LOW72GuardedTrancheVault code)`: control branches only, after the FROZEN CODE branch of the same test has finished.

### Output of the run
```
Ran 3 tests for test/LOW72SiblingQuotes.t.sol:LOW72SiblingQuotesTest
[PASS] test_1_QuoteTable_WipedJunior_NoOwnerAction() (gas: 10526956)
Logs:
  STEP 0 healthy
    [0 healthy] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 199992474809 199991476431 998378
    [0 healthy] RECEIVER value / RECEIVER shares 0 0
    [0 healthy] Senior principal / Senior high-water mark / Junior principal 800001001588 800001001588 200000998412
    [0 healthy] unassigned assets / Junior share supply 0 200317299120011
    [0 healthy] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 10016241826614
    [0 healthy] convertToShares(10,000 USDG) 10016241826614
    [0 healthy] quoteBootstrapDeposit(10,000 USDG) 10016241826614
    [0 healthy] estimateMintAssets(whole Junior supply) (USDG atoms) 199992474811
  STEP 1 TRADER_A opens LONG 3,900 lots at 1.00 (keeper executes); basket 1.00 -> 0.35; reconcile
    [1 after loss] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [1 after loss] RECEIVER value / RECEIVER shares 0 0
    [1 after loss] Senior principal / Senior high-water mark / Junior principal 746888867698 799615626457 0
    [1 after loss] unassigned assets / Junior share supply 0 200317299120011
  WIPED STATE, unmodified contracts, no owner action
    [1 wiped, frozen code] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 0
    [1 wiped, frozen code] convertToShares(10,000 USDG) 2003172991210110000000000
    [1 wiped, frozen code] quoteBootstrapDeposit(10,000 USDG) 2003172991210110000000000
    [1 wiped, frozen code] estimateMintAssets(whole Junior supply) (USDG atoms) 1
  CONTROL: the same guard on the three siblings
    [1 wiped, guard on siblings] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 0
    [1 wiped, guard on siblings] convertToShares(10,000 USDG) 0
    [1 wiped, guard on siblings] quoteBootstrapDeposit(10,000 USDG) 0
    [1 wiped, guard on siblings] estimateMintAssets(whole Junior supply) (USDG atoms) 0

[PASS] test_2_AssignmentMintsAtTheUnguardedQuote_RealizedLoss() (gas: 30014349)
Logs:
  STEP 0 healthy
    [0 healthy] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 199992474809 199991476431 998378
    [0 healthy] RECEIVER value / RECEIVER shares 0 0
    [0 healthy] Senior principal / Senior high-water mark / Junior principal 800001001588 800001001588 200000998412
    [0 healthy] unassigned assets / Junior share supply 0 200317299120011
    [0 healthy] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 10016241826614
    [0 healthy] convertToShares(10,000 USDG) 10016241826614
    [0 healthy] quoteBootstrapDeposit(10,000 USDG) 10016241826614
    [0 healthy] estimateMintAssets(whole Junior supply) (USDG atoms) 199992474811
  STEP 1 TRADER_A opens LONG 3,900 lots at 1.00 (keeper executes); basket 1.00 -> 0.35; reconcile
    [1 after loss] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [1 after loss] RECEIVER value / RECEIVER shares 0 0
    [1 after loss] Senior principal / Senior high-water mark / Junior principal 746888867698 799615626457 0
    [1 after loss] unassigned assets / Junior share supply 0 200317299120011
  STEP 2 TRADER_A closes at 0.35: the Junior loss is realized before any owner action
    [2 after TRADER_A close] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [2 after TRADER_A close] RECEIVER value / RECEIVER shares 0 0
    [2 after TRADER_A close] Senior principal / Senior high-water mark / Junior principal 746476957682 799215819146 0
    [2 after TRADER_A close] unassigned assets / Junior share supply 0 200317299120011
  STEP 3 owner recapitalizes 100,000 USDG (SECURITY.md:1509 step 4); reconcile
    [3 after recap] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [3 after recap] RECEIVER value / RECEIVER shares 0 0
    [3 after recap] Senior principal / Senior high-water mark / Junior principal 798816211738 798816211738 0
    [3 after recap] unassigned assets / Junior share supply 47261138536 200317299120011
    [3 wiped, frozen code] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 0
    [3 wiped, frozen code] convertToShares(10,000 USDG) 2003172991210110000000000
    [3 wiped, frozen code] quoteBootstrapDeposit(10,000 USDG) 2003172991210110000000000
    [3 wiped, frozen code] estimateMintAssets(whole Junior supply) (USDG atoms) 1
  FROZEN CODE step 4: owner assignUnassignedAssets; the mint is priced by quoteBootstrapDeposit
    [V4 frozen code, after assign] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [V4 frozen code, after assign] RECEIVER value / RECEIVER shares 47261138536 9467223624915451899379896
    [V4 frozen code, after assign] Senior principal / Senior high-water mark / Junior principal 798816211738 798816211738 47261138536
    [V4 frozen code, after assign] unassigned assets / Junior share supply 0 9467223625115769198499907
    RECEIVER share of Junior supply (1e18 = 100%) 999999999978840966
  FROZEN CODE step 5: TRADER_B opens LONG 1,000 lots at 0.35; basket 0.35 -> 0.40; TRADER_B closes
    [V5 frozen code, after later revenue] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 1 1 0
    [V5 frozen code, after later revenue] RECEIVER value / RECEIVER shares 52261082700 9467223624915451899379896
    [V5 frozen code, after later revenue] Senior principal / Senior high-water mark / Junior principal 798424095611 798424095611 52261082701
    [V5 frozen code, after later revenue] unassigned assets / Junior share supply 0 9467223625115769198499907
  GUARD-LATE step 4: guard installed on the siblings; the same owner call reverts
    [L3 wiped, guard on siblings] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 0
    [L3 wiped, guard on siblings] convertToShares(10,000 USDG) 0
    [L3 wiped, guard on siblings] quoteBootstrapDeposit(10,000 USDG) 0
    [L3 wiped, guard on siblings] estimateMintAssets(whole Junior supply) (USDG atoms) 0
    [L4 guard, after reverted assign] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [L4 guard, after reverted assign] RECEIVER value / RECEIVER shares 0 0
    [L4 guard, after reverted assign] Senior principal / Senior high-water mark / Junior principal 798816211738 798816211738 0
    [L4 guard, after reverted assign] unassigned assets / Junior share supply 47261138536 200317299120011
  GUARD-LATE step 5: same TRADER_B round trip
    [L5 guard, after later revenue] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 5007239290 5007214294 24996
    [L5 guard, after later revenue] RECEIVER value / RECEIVER shares 0 0
    [L5 guard, after later revenue] Senior principal / Senior high-water mark / Junior principal 798416804133 798416804133 5007239290
    [L5 guard, after later revenue] unassigned assets / Junior share supply 47261138536 200317299120011
  GUARD-LATE step 6: owner assigns the same unassigned amount at the positive Junior NAV
    [L6 guard, after late assign] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 5007239290 5007214294 24996
    [L6 guard, after late assign] RECEIVER value / RECEIVER shares 47261138535 1890707248988842
    [L6 guard, after late assign] Senior principal / Senior high-water mark / Junior principal 798416804133 798416804133 52268377826
    [L6 guard, after late assign] unassigned assets / Junior share supply 0 2091024548108853
  GUARD-EARLY step 4: guard installed; assign reverts; TRADER_B opens (its fees reach Junior)
    Junior deposit-pricing assets after the TRADER_B open (USDG atoms) 7239290
    [E4 guard, after TRADER_B open] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 276708449931092699
    [E4 guard, after TRADER_B open] convertToShares(10,000 USDG) 276708449931092699
    [E4 guard, after TRADER_B open] quoteBootstrapDeposit(10,000 USDG) 276708449931092699
    [E4 guard, after TRADER_B open] estimateMintAssets(whole Junior supply) (USDG atoms) 7239291
  GUARD-EARLY step 5: owner assigns now (guard condition false: pricing assets > 0)
    [E5 guard, after assign at positive NAV] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 7239290 7239254 36
    [E5 guard, after assign at positive NAV] RECEIVER value / RECEIVER shares 47261138535 1307755638627519172
    [E5 guard, after assign at positive NAV] Senior principal / Senior high-water mark / Junior principal 798816211738 798816211738 47268377826
    [E5 guard, after assign at positive NAV] unassigned assets / Junior share supply 0 1307955955926639183
    RECEIVER share of Junior supply (1e18 = 100%) 999846847060703895
    [E6 guard, after later revenue] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 8003938 8003899 39
    [E6 guard, after later revenue] RECEIVER value / RECEIVER shares 52253082815 1307755638627519172
    [E6 guard, after later revenue] Senior principal / Senior high-water mark / Junior principal 798424091560 798424091560 52261086754
    [E6 guard, after later revenue] unassigned assets / Junior share supply 0 1307955955926639183
  SUMMARY (realized loss, later revenue)    columns: FROZEN CODE / GUARD-LATE / GUARD-EARLY
    PRE-WIPE holders value right after the assignment 0 0 7239290
    PRE-WIPE holders value final 1 5007239290 8003938
    RECEIVER value final 52261082700 47261138535 52253082815
    Senior principal final 798424095611 798416804133 798424091560
    Junior principal final 52261082701 52268377826 52261086754
    later Junior revenue (growth of Junior principal): FROZEN CODE / GUARD-LATE 4999944165 5007239290
    FROZEN CODE: of that revenue, RECEIVER value grows by / PRE-WIPE holders value grows by 4999944164 1
    PRE-WIPE holders final, GUARD-LATE minus FROZEN CODE / GUARD-LATE minus GUARD-EARLY 5007239289 4999235352
    PRE-WIPE holders final, GUARD-EARLY minus FROZEN CODE 8003937

[PASS] test_3_Sensitivity_UnrealizedLossThenMarkReversal() (gas: 17794576)
Logs:
  STEP 0 healthy
    [0 healthy] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 199992474809 199991476431 998378
    [0 healthy] RECEIVER value / RECEIVER shares 0 0
    [0 healthy] Senior principal / Senior high-water mark / Junior principal 800001001588 800001001588 200000998412
    [0 healthy] unassigned assets / Junior share supply 0 200317299120011
    [0 healthy] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix] 10016241826614
    [0 healthy] convertToShares(10,000 USDG) 10016241826614
    [0 healthy] quoteBootstrapDeposit(10,000 USDG) 10016241826614
    [0 healthy] estimateMintAssets(whole Junior supply) (USDG atoms) 199992474811
  STEP 1 TRADER_A opens LONG 3,900 lots at 1.00 (keeper executes); basket 1.00 -> 0.35; reconcile
    [1 after loss] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [1 after loss] RECEIVER value / RECEIVER shares 0 0
    [1 after loss] Senior principal / Senior high-water mark / Junior principal 746888867698 799615626457 0
    [1 after loss] unassigned assets / Junior share supply 0 200317299120011
  STEP 2 owner recapitalizes 100,000 USDG while TRADER_A is still open; reconcile
    [2 after recap] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [2 after recap] RECEIVER value / RECEIVER shares 0 0
    [2 after recap] Senior principal / Senior high-water mark / Junior principal 799215819145 799215819145 0
    [2 after recap] unassigned assets / Junior share supply 47273241241 200317299120011
  FROZEN CODE step 3: owner assigns to Junior; basket 0.35 -> 1.00; TRADER_A closes; reconcile
    [V3 frozen code, after assign] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 0 0 0
    [V3 frozen code, after assign] RECEIVER value / RECEIVER shares 47273241241 9469648006093110254814651
    [V3 frozen code, after assign] Senior principal / Senior high-water mark / Junior principal 799215819145 799215819145 47273241241
    [V3 frozen code, after assign] unassigned assets / Junior share supply 0 9469648006293427553934662
    RECEIVER share of Junior supply (1e18 = 100%) 999999999978846383
    [V4 frozen code, after mark reversal] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 6 6 0
    [V4 frozen code, after mark reversal] RECEIVER value / RECEIVER shares 300230655664 9469648006093110254814651
    [V4 frozen code, after mark reversal] Senior principal / Senior high-water mark / Junior principal 798823506861 798823506861 300230655670
    [V4 frozen code, after mark reversal] unassigned assets / Junior share supply 0 9469648006293427553934662
  GUARD-EARLY step 3: guard installed; assign reverts; TRADER_B 1,000-lot round trip at 0.35 (fees only)
    Junior deposit-pricing assets after the fee-only round trip (USDG atoms) 14002184
  GUARD-EARLY step 4: owner assigns at that positive NAV; basket 0.35 -> 1.00; TRADER_A closes
    [E4 guard, after assign at positive NAV] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 14002184 14002115 69
    [E4 guard, after assign at positive NAV] RECEIVER value / RECEIVER shares 47273241240 676297878230655448
    [E4 guard, after assign at positive NAV] Senior principal / Senior high-water mark / Junior principal 799215819145 799215819145 47287243425
    [E4 guard, after assign at positive NAV] unassigned assets / Junior share supply 0 676498195529775459
    RECEIVER share of Junior supply (1e18 = 100%) 999703890859659810
    [E5 guard, after mark reversal] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed 88905191 88904748 443
    [E5 guard, after mark reversal] RECEIVER value / RECEIVER shares 300155769621 676297878230655448
    [E5 guard, after mark reversal] Senior principal / Senior high-water mark / Junior principal 798823498755 798823498755 300244674813
    [E5 guard, after mark reversal] unassigned assets / Junior share supply 0 676498195529775459
  SUMMARY (unrealized loss, mark reversal)    columns: FROZEN CODE / GUARD-EARLY
    PRE-WIPE holders value right after the assignment 0 14002184
    PRE-WIPE holders value final 6 88905191
    RECEIVER value final 300230655664 300155769621
    Senior principal final 798823506861 798823498755
    PRE-WIPE holders final, GUARD-EARLY minus FROZEN CODE 88905185

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 36.96ms (61.79ms CPU time)

Ran 1 test suite in 38.75ms (36.96ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### Full test code (`packages/perps/test/LOW72SiblingQuotes.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Incomplete fix of Cecuro LOW-72 ("Pending depositor can receive impossible share quotes for a wiped tranche").
//
// The fix (commit d757d0b1) added the wiped-tranche guard to TrancheVault.estimateDepositShares only
// (TrancheVault.sol:1366-1368). The sibling quotes convertToShares (:515-519), estimateMintAssets (:1372-1376) and
// quoteBootstrapDeposit (:1392-1396) still price against zero deposit-pricing assets after a terminal Junior wipe.
// HousePool.assignUnassignedAssets (HousePool.sol:660-704, owner) mints Junior shares at quoteBootstrapDeposit
// (HousePool.sol:685).
//
// Parties measured at every step:
//   PRE-WIPE holders = the Junior LP plus the Junior seed (held by the owner contract of this fixture),
//   RECEIVER         = the account the owner names in assignUnassignedAssets,
//   Senior           = pool.seniorPrincipal() (all Senior holders).
//
// Tests:
//   test_1_QuoteTable_WipedJunior_NoOwnerAction
//       Trading alone wipes Junior. With no owner action, the guarded quote returns 0 and the three sibling quotes
//       return the zero-pricing values. Control: with the same guard on the siblings, all four return 0.
//   test_2_AssignmentMintsAtTheUnguardedQuote_RealizedLoss
//       The loss is realized (TRADER_A closes) before any owner action. The owner recapitalizes; the remainder is
//       unassigned. Branch FROZEN CODE (unmodified contracts): the owner's assignment mints exactly
//       quoteBootstrapDeposit(unassigned). Later Junior revenue = TRADER_B's realized loss.
//       Branch GUARD-LATE (control): same guard on the siblings; the same owner call reverts; the owner assigns after
//       the later revenue. Branch GUARD-EARLY (control): the owner assigns as soon as one trade's fees make Junior
//       pricing assets positive (the guard condition `pricingAssets == 0` is then false).
//   test_3_Sensitivity_UnrealizedLossThenMarkReversal
//       TRADER_A stays open; the basket returns to 1.00 after the assignment. Branches FROZEN CODE and GUARD-EARLY.
//
// Run (repository root, commit a82d4fefb02768e5b3bddcc97dee371eee24fea8; this file and LOW72GuardedTrancheVault.sol in
// packages/perps/test/):
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/LOW72SiblingQuotes.t.sol -vv
// No env vars or RPC. Fixture: the repository's BasePerpTest. Mocks: MockPyth and the USDG mock only (out of scope).
//
// Role map: this test contract is the deployer and therefore the OWNER of every perps contract (as in the
// repository's own tests). KEEPER, TRADER_A, TRADER_B, JUNIOR_LP and SENIOR_LP are untrusted accounts; vm.prank only
// sets msg.sender for their own public calls. RECEIVER makes no call. No protocol address is impersonated and
// vm.store is not used. vm.etch is used ONLY in the control branches, to install LOW72GuardedTrancheVault (a copy of
// TrancheVault with the same guard on the three siblings) at the Junior vault address. Every FROZEN CODE branch runs
// before any vm.etch, on unmodified in-scope contracts.

import {LOW72GuardedTrancheVault} from "./LOW72GuardedTrancheVault.sol";
import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

contract LOW72SiblingQuotesTest is BasePerpTest {

    address internal constant KEEPER = address(0x0E0E1); // any caller; keepers are untrusted (program page)
    address internal constant TRADER_A = address(0xA001);
    address internal constant TRADER_B = address(0xB001);
    address internal constant JUNIOR_LP = address(0x1A01); // pre-wipe Junior holder
    address internal constant SENIOR_LP = address(0x5A01); // Senior holder
    address internal constant NEW_LP = address(0x1A03); // would-be Junior entrant after the wipe
    address internal constant RECEIVER = address(0xB007); // account named by the owner in assignUnassignedAssets

    uint256 internal constant LOT = 1e20;
    uint256 internal constant MON = 1_710_151_200; // Mon 2024-03-11 10:00 UTC (market live)
    uint256 internal constant P0 = 1e8;
    uint256 internal constant Q = 10_000e6; // quote size used in the LOW-72 audit item
    uint256 internal constant VIRTUAL_SHARES = 1000; // TrancheVault.sol:36

    uint256 internal nonce;

    struct Parties {
        uint256 preWipe; // Junior LP + Junior seed, redeemable value (convertToAssets)
        uint256 juniorLp;
        uint256 juniorSeed;
        uint256 receiver; // redeemable value
        uint256 receiverShares;
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 unassigned;
        uint256 juniorSupply;
    }

    // Release risk parameters (DEPLOYMENT.md: MM 10 bps, IM 20 bps, frozen-market margin 300 bps).
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000); // release Senior cap: 40M USDG and 80 % of the pool
        _warpTo(MON);
        _pushMark(P0);
        uint256 a = _requestDeposit(JUNIOR_LP, juniorVault, 200_000e6);
        _warpTo(pool.lpEpochStart(a));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(JUNIOR_LP, juniorVault, a);
        _warpTo(_now() + 60);
        _pushMark(P0);
        uint256 s = _requestDeposit(SENIOR_LP, seniorVault, 800_000e6);
        _warpTo(pool.lpEpochStart(s));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(SENIOR_LP, seniorVault, s);
        _fundTrader(TRADER_A, 2_000_000e6);
        _fundTrader(TRADER_B, 2_000_000e6);
    }

    // ------------------------------------------------------------------ helpers

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _warpTo(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _liveData(
        uint256 p
    ) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(p)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime())
            );
        }
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _pushMark(
        uint256 p
    ) internal {
        bytes[] memory d = _liveData(p);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _atomicSettle(
        uint256 p
    ) internal {
        bytes[] memory d = _liveData(p);
        vm.prank(KEEPER);
        router.settleLpEpoch(d);
    }

    function _requestDeposit(
        address lp,
        TrancheVault v,
        uint256 amt
    ) internal returns (uint256 id) {
        settlementToken.mint(lp, amt);
        vm.startPrank(lp);
        settlementToken.approve(address(v), amt);
        id = v.requestDeposit(amt, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        address lp,
        TrancheVault v,
        uint256 id
    ) internal returns (uint256 shares) {
        uint256 amt = v.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = v.claimDeposit(id, amt, lp, lp);
    }

    function _requestRedeem(
        address lp,
        TrancheVault v,
        uint256 shares
    ) internal returns (uint256 id) {
        vm.prank(lp);
        id = v.requestRedeem(shares, lp, lp);
    }

    function _request(
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        nonce += 1;
        r.clientOrderId = keccak256(abi.encode("LOW72", nonce));
        r.side = side;
        r.sizeDelta = lots * LOT;
        r.marginDelta = margin;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now() + 60),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev A trader commits an order; the keeper executes it at mark `p`.
    function _trade(
        address who,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose,
        uint256 p
    ) internal {
        _warpTo(_now() + 1);
        _pushMark(p);
        OrderV3Types.OrderRequest memory r = _request(side, lots, margin, isClose);
        vm.prank(who);
        uint64 id = router.commitOrder(r);
        bytes[] memory upd = _mockPythUpdateData(p);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(id, upd);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "trade executed");
    }

    /// @dev Any matured LP work forces the reconcile inside settlement; a tiny Senior redemption provides it
    ///      (same schedule in every branch).
    function _reconcileNow(
        uint256 p
    ) internal {
        uint256 rid = _requestRedeem(SENIOR_LP, seniorVault, seniorVault.balanceOf(SENIOR_LP) / 2000);
        _warpTo(pool.lpEpochStart(rid) + 10);
        _atomicSettle(p);
    }

    /// @dev OWNER call: CfdEngine.recapitalizePool (the test contract is the deployed owner).
    function _recap(
        uint256 amt
    ) internal {
        settlementToken.mint(address(this), amt);
        settlementToken.approve(address(engine), amt);
        engine.recapitalizePool(amt);
    }

    /// @dev OWNER call: HousePool.assignUnassignedAssets to the Junior tranche.
    function _assignJunior() internal {
        _pushMark(engine.lastMarkPrice()); // assignUnassignedAssets requires a fresh mark (HousePool.sol:670)
        pool.assignUnassignedAssets(false, RECEIVER);
    }

    function _parties(
        string memory tag
    ) internal view returns (Parties memory c) {
        c.juniorLp = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP));
        c.juniorSeed = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        c.preWipe = c.juniorLp + c.juniorSeed;
        c.receiverShares = juniorVault.balanceOf(RECEIVER);
        c.receiver = juniorVault.convertToAssets(c.receiverShares);
        c.seniorPrincipal = pool.seniorPrincipal();
        c.juniorPrincipal = pool.juniorPrincipal();
        c.unassigned = pool.unassignedAssets();
        c.juniorSupply = juniorVault.totalSupply();
        console.log(
            string.concat("  [", tag, "] PRE-WIPE holders value (Junior LP + seed) / Junior LP / seed"),
            c.preWipe,
            c.juniorLp,
            c.juniorSeed
        );
        console.log(string.concat("  [", tag, "] RECEIVER value / RECEIVER shares"), c.receiver, c.receiverShares);
        console.log(
            string.concat("  [", tag, "] Senior principal / Senior high-water mark / Junior principal"),
            c.seniorPrincipal,
            pool.seniorHighWaterMark(),
            c.juniorPrincipal
        );
        console.log(string.concat("  [", tag, "] unassigned assets / Junior share supply"), c.unassigned, c.juniorSupply);
    }

    /// @dev View calls only; they need no role.
    function _logQuotes(
        string memory tag
    ) internal view {
        uint256 supply = juniorVault.accruedTotalSupply();
        console.log(
            string.concat("  [", tag, "] estimateDepositShares(10,000 USDG)  [guarded by the LOW-72 fix]"),
            juniorVault.estimateDepositShares(Q)
        );
        console.log(string.concat("  [", tag, "] convertToShares(10,000 USDG)"), juniorVault.convertToShares(Q));
        console.log(
            string.concat("  [", tag, "] quoteBootstrapDeposit(10,000 USDG)"), juniorVault.quoteBootstrapDeposit(Q)
        );
        console.log(
            string.concat("  [", tag, "] estimateMintAssets(whole Junior supply) (USDG atoms)"),
            juniorVault.estimateMintAssets(supply)
        );
    }

    /// @dev CONTROL ONLY: installs the copy of TrancheVault with the guard on the three siblings at the Junior vault
    ///      address. Storage and immutables of the Junior vault are unchanged (same constructor arguments).
    function _installGuard() internal {
        LOW72GuardedTrancheVault guarded = new LOW72GuardedTrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0)
        );
        vm.etch(address(juniorVault), address(guarded).code);
    }

    // ------------------------------------------------------------------ shared path to the wiped state

    /// @dev No owner call. A trader's gain larger than Junior NAV wipes Junior principal; the supply survives.
    ///      The size of the move (1.00 -> 0.35) is only the way this test reaches the wiped state.
    function _wipe() internal {
        _warpTo(MON + 3 hours + 10 minutes);
        console.log("STEP 0 healthy");
        _parties("0 healthy");
        _logQuotes("0 healthy");
        console.log("STEP 1 TRADER_A opens LONG 3,900 lots at 1.00 (keeper executes); basket 1.00 -> 0.35; reconcile");
        _trade(TRADER_A, CfdTypes.Side.LONG, 3900, 300_000e6, false, P0);
        _warpTo(_now() + 60);
        _reconcileNow(0.35e8);
        _parties("1 after loss");
        assertEq(pool.juniorPrincipal(), 0, "Junior principal wiped");
        assertGt(juniorVault.totalSupply(), 0, "Junior supply survives");
        assertEq(juniorVault.maxRequestDeposit(NEW_LP), 0, "ordinary Junior entry closed");
    }

    // =====================================================================================================
    // 1. Quote table in the wiped state, before any owner action
    // =====================================================================================================

    function test_1_QuoteTable_WipedJunior_NoOwnerAction() public {
        _wipe();
        uint256 supply = juniorVault.accruedTotalSupply();

        console.log("WIPED STATE, unmodified contracts, no owner action");
        _logQuotes("1 wiped, frozen code");
        assertEq(juniorVault.estimateDepositShares(Q), 0, "guarded quote (LOW-72 fix) returns 0");
        assertEq(juniorVault.convertToShares(Q), Q * (supply + VIRTUAL_SHARES), "convertToShares: zero-pricing quote");
        assertEq(
            juniorVault.quoteBootstrapDeposit(Q), Q * (supply + VIRTUAL_SHARES), "quoteBootstrapDeposit: zero-pricing quote"
        );
        assertEq(juniorVault.estimateMintAssets(supply), 1, "estimateMintAssets: whole supply for 1 atom");

        console.log("CONTROL: the same guard on the three siblings");
        _installGuard();
        _logQuotes("1 wiped, guard on siblings");
        assertEq(juniorVault.estimateDepositShares(Q), 0, "guarded estimateDepositShares");
        assertEq(juniorVault.convertToShares(Q), 0, "guarded convertToShares");
        assertEq(juniorVault.quoteBootstrapDeposit(Q), 0, "guarded quoteBootstrapDeposit");
        assertEq(juniorVault.estimateMintAssets(supply), 0, "guarded estimateMintAssets");
    }

    // =====================================================================================================
    // 2. The owner's assignment mints at the unguarded quote (realized loss, then later revenue)
    // =====================================================================================================

    function test_2_AssignmentMintsAtTheUnguardedQuote_RealizedLoss() public {
        _wipe();
        console.log("STEP 2 TRADER_A closes at 0.35: the Junior loss is realized before any owner action");
        _trade(TRADER_A, CfdTypes.Side.LONG, 3900, 0, true, 0.35e8);
        _reconcileNow(0.35e8);
        _parties("2 after TRADER_A close");
        console.log("STEP 3 owner recapitalizes 100,000 USDG (SECURITY.md:1509 step 4); reconcile");
        _recap(100_000e6);
        _reconcileNow(0.35e8);
        Parties memory c3 = _parties("3 after recap");
        assertEq(c3.juniorPrincipal, 0, "Junior still wiped");
        assertGt(c3.unassigned, 0, "recap remainder is unassigned");
        _logQuotes("3 wiped, frozen code");
        assertEq(juniorVault.estimateDepositShares(Q), 0, "guarded quote (LOW-72 fix) returns 0");
        assertEq(juniorVault.estimateMintAssets(juniorVault.accruedTotalSupply()), 1, "whole supply for 1 atom");
        uint256 snap = vm.snapshotState();

        // ---------------- FROZEN CODE: unmodified contracts; the owner assigns at zero pricing assets
        console.log("FROZEN CODE step 4: owner assignUnassignedAssets; the mint is priced by quoteBootstrapDeposit");
        uint256 quoted = juniorVault.quoteBootstrapDeposit(c3.unassigned);
        _pushMark(engine.lastMarkPrice()); // assignUnassignedAssets requires a fresh mark (HousePool.sol:670)
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        pool.assignUnassignedAssets(true, RECEIVER); // OWNER call; Senior target: no capacity while Junior principal is 0
        pool.assignUnassignedAssets(false, RECEIVER); // OWNER call; Junior target
        Parties memory v4 = _parties("V4 frozen code, after assign");
        assertEq(v4.receiverShares, quoted, "minted exactly quoteBootstrapDeposit(unassigned)");
        assertEq(quoted, c3.unassigned * (c3.juniorSupply + VIRTUAL_SHARES), "= amount x (surviving supply + 1000)");
        assertEq(v4.receiver, c3.unassigned, "receiver's value equals the assigned amount");
        assertEq(v4.seniorPrincipal, c3.seniorPrincipal, "Senior unchanged by the assignment");
        console.log("  RECEIVER share of Junior supply (1e18 = 100%)", v4.receiverShares * 1e18 / v4.juniorSupply);
        console.log("FROZEN CODE step 5: TRADER_B opens LONG 1,000 lots at 0.35; basket 0.35 -> 0.40; TRADER_B closes");
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 50_000e6, false, 0.35e8);
        _warpTo(_now() + 60);
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 0, true, 0.4e8);
        _reconcileNow(0.4e8);
        Parties memory v5 = _parties("V5 frozen code, after later revenue");

        vm.revertToState(snap);

        // ---------------- GUARD-LATE (control): guard on the siblings; the owner assigns after the later revenue
        console.log("GUARD-LATE step 4: guard installed on the siblings; the same owner call reverts");
        _installGuard();
        _logQuotes("L3 wiped, guard on siblings");
        assertEq(juniorVault.quoteBootstrapDeposit(Q), 0, "guarded quoteBootstrapDeposit");
        assertEq(juniorVault.convertToShares(Q), 0, "guarded convertToShares");
        assertEq(juniorVault.estimateMintAssets(juniorVault.accruedTotalSupply()), 0, "guarded estimateMintAssets");
        _pushMark(engine.lastMarkPrice());
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        pool.assignUnassignedAssets(true, RECEIVER); // OWNER call
        vm.expectRevert(IHousePool.HousePool__BootstrapSharesZero.selector);
        pool.assignUnassignedAssets(false, RECEIVER); // OWNER call: the guarded quote returns 0 (HousePool.sol:686-688)
        Parties memory l4 = _parties("L4 guard, after reverted assign");
        console.log("GUARD-LATE step 5: same TRADER_B round trip");
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 50_000e6, false, 0.35e8);
        _warpTo(_now() + 60);
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 0, true, 0.4e8);
        _reconcileNow(0.4e8);
        Parties memory l5 = _parties("L5 guard, after later revenue");
        console.log("GUARD-LATE step 6: owner assigns the same unassigned amount at the positive Junior NAV");
        _assignJunior();
        Parties memory l6 = _parties("L6 guard, after late assign");

        vm.revertToState(snap);

        // ---------------- GUARD-EARLY (control): guard installed; the owner assigns at the first positive Junior NAV
        console.log("GUARD-EARLY step 4: guard installed; assign reverts; TRADER_B opens (its fees reach Junior)");
        _installGuard();
        _pushMark(engine.lastMarkPrice());
        vm.expectRevert(IHousePool.HousePool__BootstrapSharesZero.selector);
        pool.assignUnassignedAssets(false, RECEIVER); // OWNER call
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 50_000e6, false, 0.35e8);
        (, uint256 pricingAfterOpen) = pool.getPendingDepositTrancheState();
        console.log("  Junior deposit-pricing assets after the TRADER_B open (USDG atoms)", pricingAfterOpen);
        _logQuotes("E4 guard, after TRADER_B open");
        console.log("GUARD-EARLY step 5: owner assigns now (guard condition false: pricing assets > 0)");
        _assignJunior();
        Parties memory e5 = _parties("E5 guard, after assign at positive NAV");
        console.log("  RECEIVER share of Junior supply (1e18 = 100%)", e5.receiverShares * 1e18 / e5.juniorSupply);
        _warpTo(_now() + 60);
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 0, true, 0.4e8);
        _reconcileNow(0.4e8);
        Parties memory e6 = _parties("E6 guard, after later revenue");

        console.log("SUMMARY (realized loss, later revenue)    columns: FROZEN CODE / GUARD-LATE / GUARD-EARLY");
        console.log("  PRE-WIPE holders value right after the assignment", v4.preWipe, l4.preWipe, e5.preWipe);
        console.log("  PRE-WIPE holders value final", v5.preWipe, l6.preWipe, e6.preWipe);
        console.log("  RECEIVER value final", v5.receiver, l6.receiver, e6.receiver);
        console.log("  Senior principal final", v5.seniorPrincipal, l6.seniorPrincipal, e6.seniorPrincipal);
        console.log("  Junior principal final", v5.juniorPrincipal, l6.juniorPrincipal, e6.juniorPrincipal);
        console.log(
            "  later Junior revenue (growth of Junior principal): FROZEN CODE / GUARD-LATE",
            v5.juniorPrincipal - v4.juniorPrincipal,
            l5.juniorPrincipal - l4.juniorPrincipal
        );
        console.log(
            "  FROZEN CODE: of that revenue, RECEIVER value grows by / PRE-WIPE holders value grows by",
            v5.receiver - v4.receiver,
            v5.preWipe - v4.preWipe
        );
        console.log(
            "  PRE-WIPE holders final, GUARD-LATE minus FROZEN CODE / GUARD-LATE minus GUARD-EARLY",
            l6.preWipe - v5.preWipe,
            l6.preWipe - e6.preWipe
        );
        console.log("  PRE-WIPE holders final, GUARD-EARLY minus FROZEN CODE", e6.preWipe - v5.preWipe);

        assertEq(c3.preWipe, 0, "pre-wipe holders hold no value before any owner action");
        assertEq(v4.preWipe, 0, "FROZEN CODE: still 0 right after the assign (nothing they still had is taken)");
        assertGt(l6.preWipe, 1000e6, "GUARD-LATE: pre-wipe holders receive the later revenue");
        assertLt(v5.preWipe, 1e6, "FROZEN CODE: pre-wipe holders receive < 1 USDG of it");
        assertGt(v5.receiver, l6.receiver, "FROZEN CODE: receiver holds the later revenue on top of the assigned amount");
        assertGt(pricingAfterOpen, 0, "one trade's fees make Junior pricing assets positive");
        assertGt(
            e5.receiverShares * 1e18 / e5.juniorSupply, 0.999e18, "GUARD-EARLY: receiver still > 99.9 % of supply"
        );
        assertLt(e6.preWipe - v5.preWipe, 10e6, "the guard changes the pre-wipe holders' result by < 10 USDG here");
    }

    // =====================================================================================================
    // 3. Sensitivity: unrealized loss, then mark reversal after the assignment
    // =====================================================================================================

    function test_3_Sensitivity_UnrealizedLossThenMarkReversal() public {
        _wipe();
        console.log("STEP 2 owner recapitalizes 100,000 USDG while TRADER_A is still open; reconcile");
        _recap(100_000e6);
        _reconcileNow(0.35e8);
        Parties memory c2 = _parties("2 after recap");
        assertEq(c2.juniorPrincipal, 0, "Junior still wiped");
        assertGt(c2.unassigned, 0, "recap remainder is unassigned");
        assertEq(c2.preWipe, 0, "pre-wipe holders hold no value before the assignment");
        uint256 snap = vm.snapshotState();

        console.log("FROZEN CODE step 3: owner assigns to Junior; basket 0.35 -> 1.00; TRADER_A closes; reconcile");
        _assignJunior();
        Parties memory v3 = _parties("V3 frozen code, after assign");
        console.log("  RECEIVER share of Junior supply (1e18 = 100%)", v3.receiverShares * 1e18 / v3.juniorSupply);
        _warpTo(_now() + 60);
        _trade(TRADER_A, CfdTypes.Side.LONG, 3900, 0, true, P0);
        _reconcileNow(P0);
        Parties memory v4 = _parties("V4 frozen code, after mark reversal");

        vm.revertToState(snap);

        console.log("GUARD-EARLY step 3: guard installed; assign reverts; TRADER_B 1,000-lot round trip at 0.35 (fees only)");
        _installGuard();
        _pushMark(engine.lastMarkPrice());
        vm.expectRevert(IHousePool.HousePool__BootstrapSharesZero.selector);
        pool.assignUnassignedAssets(false, RECEIVER); // OWNER call
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 50_000e6, false, 0.35e8);
        _trade(TRADER_B, CfdTypes.Side.LONG, 1000, 0, true, 0.35e8);
        (, uint256 pricingAfterRoundTrip) = pool.getPendingDepositTrancheState();
        console.log("  Junior deposit-pricing assets after the fee-only round trip (USDG atoms)", pricingAfterRoundTrip);
        console.log("GUARD-EARLY step 4: owner assigns at that positive NAV; basket 0.35 -> 1.00; TRADER_A closes");
        _assignJunior();
        Parties memory e4 = _parties("E4 guard, after assign at positive NAV");
        console.log("  RECEIVER share of Junior supply (1e18 = 100%)", e4.receiverShares * 1e18 / e4.juniorSupply);
        _warpTo(_now() + 60);
        _trade(TRADER_A, CfdTypes.Side.LONG, 3900, 0, true, P0);
        _reconcileNow(P0);
        Parties memory e5 = _parties("E5 guard, after mark reversal");

        console.log("SUMMARY (unrealized loss, mark reversal)    columns: FROZEN CODE / GUARD-EARLY");
        console.log("  PRE-WIPE holders value right after the assignment", v3.preWipe, e4.preWipe);
        console.log("  PRE-WIPE holders value final", v4.preWipe, e5.preWipe);
        console.log("  RECEIVER value final", v4.receiver, e5.receiver);
        console.log("  Senior principal final", v4.seniorPrincipal, e5.seniorPrincipal);
        console.log("  PRE-WIPE holders final, GUARD-EARLY minus FROZEN CODE", e5.preWipe - v4.preWipe);

        assertEq(v3.preWipe, 0, "FROZEN CODE: still 0 right after the assign");
        assertGt(pricingAfterRoundTrip, 0, "fee-only round trip makes Junior pricing assets positive");
        assertLt(v4.preWipe, 1e6, "FROZEN CODE: pre-wipe holders keep < 1 USDG after the reversal");
        assertLt(e5.preWipe - v4.preWipe, 100e6, "the guard changes the pre-wipe holders' result by < 100 USDG here");
    }

}
```

