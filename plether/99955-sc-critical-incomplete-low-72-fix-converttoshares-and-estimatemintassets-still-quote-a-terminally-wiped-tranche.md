# \#99955 [SC-Critical] Incomplete LOW-72 fix: `convertToShares` and `estimateMintAssets` still quote a terminally wiped tranche

**Submitted on Nov 15th 2026 at 21:49:05 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99955
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The Cecuro audit's LOW-72 found that a deposit quote presents a wiped tranche as convertible into shares, although a
pending request in that state can only be refunded. Its fix guards one view, `TrancheVault.estimateDepositShares`. The
two public sibling views that price deposits from the same input, `convertToShares` and `estimateMintAssets`, were left
unguarded. In the exact state the fix targets, the vault quotes 100 USDG as one hundred million times the entire
surviving share supply, and quotes minting that whole supply at one USDG atom.

### Vulnerability Details

#### Background: a terminally wiped tranche

Each tranche (`TrancheVault`, Senior and Junior) prices deposits from `_depositPricingAssets()`, the tranche's projected
claim on HousePool value. Deposits are asynchronous: a depositor requests, the request matures at an LP epoch, and
HousePool activates it at settlement. When losses wipe a tranche, its projected pricing assets reach zero while its share
supply survives. From then on new requests are refused (`maxRequestDeposit` returns zero, `TrancheVault.sol:1313-1326`),
HousePool defers activation of matured deposits (`HousePool.sol:917`, `:932`), and a pending request can only be
cancelled for a full refund through the projected-wipe branch (`TrancheVault.sol:851-855`, `:1643-1647`).

#### The prior finding and its fix

Cecuro audit (2026-10-26), LOW-72, "Pending depositor can receive impossible share quotes for a wiped tranche", listed
as Acknowledged with a code fix. Its comment reads:

> LOW-72 - Addressed. estimateDepositShares returns zero when projected deposit-pricing assets are zero and shares
> survive, including before stored principal has been reconciled. [...] The guard is specific to a projected wipe [...]
> Fix: https://github.com/Plether-Fi/plether-core/commit/899675594d1e043383e10252a8985bac2e5b6271

The finding's impact is that a deposit-pricing quote presents a wiped tranche request as convertible into shares,
although "the only reachable outcome is cancellation and refund".

#### The guard covers one of three deposit-pricing views

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` the fix guards `estimateDepositShares` only:

```solidity
// TrancheVault.sol:1361-1370
function estimateDepositShares(uint256 assets) public view returns (uint256) {
    uint256 pricingAssets = _depositPricingAssets();
    uint256 pricingSupply = accruedTotalSupply();
    if (pricingAssets == 0 && pricingSupply != 0) {
        return 0;
    }
    return quoteDepositFromState(assets, pricingAssets, pricingSupply, 0);
}
```

Two public sibling views price against the same `_depositPricingAssets()` input with the same surviving supply and have
no such guard:

```solidity
// TrancheVault.sol:515-519
function convertToShares(uint256 assets) public view override returns (uint256) {
    return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
}

// TrancheVault.sol:1372-1376 (via _previewMintAssets, :1869-1873)
function estimateMintAssets(uint256 shares) external view returns (uint256) {
    return _previewMintAssets(shares); // _convertToAssetsUsingAssets(shares, _depositPricingAssets(), Ceil)
}
```

With projected pricing assets equal to zero and supply `S > 0`, `_convertToSharesUsingAssets` (`TrancheVault.sol:1853-1859`)
returns `assets * (S + 1000) / 1`, and `_previewMintAssets` returns `ceil(shares * 1 / (S + 1000))`.
`estimateMintAssets` is the share-denominated twin of the fixed `estimateDepositShares`, and `convertToShares` is the
ERC-4626 conversion view integrators read for the same purpose, so both carry the root cause the fix addressed.

#### Scenario

1. A tranche has surviving shares and a pending deposit request.
2. Losses wipe the tranche: projected deposit-pricing assets reach zero while supply survives. The request can never
   activate and can only be cancelled for a full refund.
3. `estimateDepositShares(assets)` returns 0, as the LOW-72 fix intends.
4. `convertToShares(assets)` returns `assets * (S + 1000)`, and `estimateMintAssets(S)` returns 1 atom.
5. Both quotes stay live after the wipe is reconciled into stored principal.

### Impact Details

Any wallet, dashboard, accounting job or integrator that values a pending or prospective deposit through
`convertToShares` or `estimateMintAssets` is shown a share conversion the vault will never deliver: in the proof of
concept, 100 USDG quoted as 100,000,000,100,000,000,000 shares against a surviving supply of 1,000,000,000,000, and the
whole supply quoted at 1 USDG atom. This is the request-state integrity failure LOW-72 describes, left open on two of
the three deposit-pricing views. No funds move, and no in-scope contract consumes these two views for a state change
(`PerpsPublicLens` reads the fixed `estimateDepositShares`).

### Severity

Low: contract fails to deliver promised returns, but doesn't lose value.

### Mitigation

Apply the projected-wipe guard used by `estimateDepositShares` to the public deposit-side conversion views, as in the
diff under "Fix check" in the Proof of Concept. `quoteBootstrapDeposit` is intentionally left unchanged, because the
owner-only bootstrap paths (`HousePool.sol:685`, `:738`) rely on a positive quote.

### References

- `packages/perps/src/TrancheVault.sol:1361-1370` (`estimateDepositShares`, the fixed view)
- `packages/perps/src/TrancheVault.sol:515-519` (`convertToShares`, unguarded)
- `packages/perps/src/TrancheVault.sol:1372-1376`, `:1869-1873` (`estimateMintAssets` / `_previewMintAssets`, unguarded)
- `packages/perps/src/TrancheVault.sol:1853-1859` (`_convertToSharesUsingAssets`)
- `packages/perps/src/TrancheVault.sol:1313-1326`, `:851-855`, `:1643-1647` (entry closed, refund-only branch)
- `packages/perps/src/HousePool.sol:917`, `:932` (activation deferred)
- `audits/plether-ai-audit-report-2026-10-26.pdf`, LOW-72 (Acknowledged; fix commit 8996755)

## Proof of Concept

One test file, added to the frozen commit (`a82d4fef`) with no other change. The output and the fix check below were
produced on a checkout of that commit with no tracked file modified and this file as the only added source. The test
extends the repository fixture `HousePoolAsyncTestBase` (`packages/perps/test/perps/shared/HousePoolTestBase.sol`),
which deploys the real HousePool, both TrancheVaults and the async deposit machinery, and uses its helpers:

- `_requestAsyncDeposit(vault, account, assets)` submits an async deposit request through the vault;
- `_refreshMarkForAsyncSettlement()` installs a fresh mark so LP settlement checks pass.

The wipe is produced exactly as the LOW-72 fix's own regression test produces it
(`test/perps/spec/lp/DepositEstimate.t.sol:37`): the pool's settlement-token balance is burned through the fixture's
mock token to model a total pool loss, while the pending deposit's vault escrow is untouched. No production source is
modified and no `vm.mockCall` is used. The test runs once for each tranche.

### Step-by-step

1. Read `convertToShares(100 USDG)` on the healthy tranche as a reference.
2. ALICE requests an async deposit of 100 USDG. The test warps to the request's LP epoch start and refreshes the mark.
3. The pool's settlement-token balance is burned (total pool loss). The pending deposit's escrow is untouched.
4. The test asserts the projected-wipe state the LOW-72 fix targets: `totalSupply() > 0`, `totalAssets() == 0`, and
   `maxRequestDeposit(ALICE) == 0`.
5. The fixed view behaves as intended: `estimateDepositShares(100 USDG) == 0`.
6. In the same state, the unguarded siblings still quote: `convertToShares(100 USDG) == 100 USDG * (supply + 1000)`,
   larger than the whole surviving supply, and `estimateMintAssets(supply) == 1` USDG atom.
7. `pool.reconcile()` writes the wipe into stored principal (principal 0) and leaves `canSettleDepositEntries()` false,
   so the request can never activate. Both sibling quotes are unchanged.
8. ALICE cancels and receives the full 100 USDG back, the only reachable outcome for a request the siblings still price
   as convertible.

### Setup

Save the file as `packages/perps/test/poc/Low72SiblingQuotes.t.sol` (its relative import expects that directory) and run
from `packages/perps`:

```
forge test --match-path test/poc/Low72SiblingQuotes.t.sol -vv
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../perps/shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice LOW-72 fix completeness. The fix guards only `estimateDepositShares`; the public sibling deposit-pricing
///         views `convertToShares` and `estimateMintAssets` still quote a wiped tranche.
/// @dev Builds the wiped state exactly as the fix's own regression test does
///      (test/perps/spec/lp/DepositEstimate.t.sol), so the only variable is which quote function is read.
contract Low72SiblingQuotesTest is HousePoolAsyncTestBase {

    address internal constant ALICE = address(0xA11CE);
    uint256 internal constant ASSETS = 100e6;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_juniorWipeLeavesSiblingQuotesLive() public {
        _assertSiblingQuotesLive(false);
    }

    function test_seniorWipeLeavesSiblingQuotesLive() public {
        _assertSiblingQuotesLive(true);
    }

    function _wipe(
        TrancheVault vault
    ) internal returns (uint256 requestId) {
        requestId = _requestAsyncDeposit(vault, ALICE, ASSETS);
        vm.warp(pool.lpEpochStart(requestId));
        _refreshMarkForAsyncSettlement();
        // Same pool-loss model as DepositEstimate.t.sol: the pending deposit's vault escrow is untouched.
        settlementToken.burn(address(pool), settlementToken.balanceOf(address(pool)));
    }

    function _assertSiblingQuotesLive(
        bool isSenior
    ) internal {
        TrancheVault vault = isSenior ? seniorVault : juniorVault;
        uint256 healthyQuote = vault.convertToShares(ASSETS);
        uint256 requestId = _wipe(vault);

        uint256 supply = vault.totalSupply();
        assertGt(supply, 0, "surviving shares");
        assertEq(vault.totalAssets(), 0, "projected wipe");
        assertEq(vault.maxRequestDeposit(ALICE), 0, "new entry is closed");

        // The fixed function suppresses the quote.
        assertEq(vault.estimateDepositShares(ASSETS), 0, "LOW-72 fix: estimateDepositShares is zero");

        // Unfixed siblings, same state, same pricing inputs.
        uint256 convertQuote = vault.convertToShares(ASSETS);
        uint256 mintCostForSupply = vault.estimateMintAssets(supply);
        emit log_named_uint("healthy convertToShares(100 USDG)", healthyQuote);
        emit log_named_uint("wiped   convertToShares(100 USDG)", convertQuote);
        emit log_named_uint("surviving share supply", supply);
        emit log_named_uint("wiped   estimateMintAssets(supply) in USDG atoms", mintCostForSupply);

        assertEq(convertQuote, ASSETS * (supply + 1000), "convertToShares still quotes the wiped tranche");
        assertGt(convertQuote, supply, "100 USDG is quoted as more shares than the whole surviving supply");
        assertEq(mintCostForSupply, 1, "minting the entire surviving supply is quoted at one USDG atom");

        // The quotes stay live after the wipe is reconciled into stored principal, while the request can only refund.
        vm.prank(address(vault));
        pool.reconcile();
        assertEq(isSenior ? pool.seniorPrincipal() : pool.juniorPrincipal(), 0);
        assertFalse(pool.canSettleDepositEntries(), "the pending request can never activate");
        assertEq(vault.estimateDepositShares(ASSETS), 0);
        assertEq(vault.convertToShares(ASSETS), convertQuote, "still quoting after reconcile");
        assertEq(vault.estimateMintAssets(supply), 1, "still quoting after reconcile");

        vm.prank(ALICE);
        assertEq(vault.cancelPendingDeposit(requestId, ALICE, ALICE), ASSETS, "only outcome is a full refund");
    }

}
```

### Output

```
Ran 2 tests for test/poc/Low72SiblingQuotes.t.sol:Low72SiblingQuotesTest
[PASS] test_juniorWipeLeavesSiblingQuotesLive() (gas: 1035932)
Logs:
  healthy convertToShares(100 USDG): 100000000000
  wiped   convertToShares(100 USDG): 100000000100000000000
  surviving share supply: 1000000000000
  wiped   estimateMintAssets(supply) in USDG atoms: 1

[PASS] test_seniorWipeLeavesSiblingQuotesLive() (gas: 1126429)
Logs:
  healthy convertToShares(100 USDG): 100000000000
  wiped   convertToShares(100 USDG): 100000000100000000000
  surviving share supply: 1000000000000
  wiped   estimateMintAssets(supply) in USDG atoms: 1

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 33.34ms (16.20ms CPU time)
```

### Fix check

With the projected-wipe guard applied to both views (diff below), both tests fail with `TrancheVault__TerminallyWiped()`.
The existing suites that exercise these views and the bootstrap path still pass under the patch:
`DepositEstimate.t.sol` (2 tests), `HousePoolBootstrap.t.sol` (25), `SeniorCapacity.t.sol` (two suites, 7 and 32) and
`JuniorMaintenanceFee.t.sol` (30), which is 5 suites and 96 tests with 0 failed. The source was restored afterwards.

```diff
     function convertToShares(
         uint256 assets
     ) public view override returns (uint256) {
-        return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
+        uint256 pricingAssets = _depositPricingAssets();
+        if (pricingAssets == 0 && accruedTotalSupply() != 0) {
+            return 0;
+        }
+        return _convertToSharesUsingAssets(assets, pricingAssets, Math.Rounding.Floor);
     }

     function estimateMintAssets(
         uint256 shares
     ) external view returns (uint256) {
+        if (_depositPricingAssets() == 0 && accruedTotalSupply() != 0) {
+            revert TrancheVault__TerminallyWiped();
+        }
         return _previewMintAssets(shares);
     }
```
