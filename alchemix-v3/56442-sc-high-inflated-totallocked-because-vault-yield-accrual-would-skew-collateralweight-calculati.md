# 56442 sc high inflated totallocked because vault yield accrual would skew collateralweight calculation

**Submitted on Oct 16th 2025 at 02:15:36 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56442
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

MYT is VaultV2 share which overtime would accrue yield. this yield more or less growing and can be used as part of the user debt payment.

but there are issue in `AlchemistV3::_addDebt` that chain into `redeem` function, which makes the `_collateralWeight` inaccurate.

## Vulnerability Details

```solidity
        // Update collateral variables
@>      uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
@>      uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

        account.rawLocked = lockedCollateral + toLock;
@>      _totalLocked += toLock;
        account.debt += amount;
        totalDebt += a
```

`toLock` which is the amount of shares (MYT) needed for debt amount is added into `_totalLocked` .

notice that this value `toLock` is recalculated using `convertDebtTokensToYield` which if the VaultV2 accruing yield, the same amount at timestamp X would result different share if done at timestamp X + Y where the yield accrued.

this ultimately create discrepancy in the real \_totalLocked, because how it is updated only with `toLock` which after many subsequent deposit it would became inflated/innacurate.

example: at timestamp X, the asset per share is 1:1, user mint 100 debt token which need 111 MYT as collateral. 111 MYT is added to `_totalLocked`. `account.rawLocked` is 111 MYT.

at timestamp X+Y, the asset per share is 1:0.9, user mint 100 debt token which need only 99.9 MYT collateral. 99.9 MYT is added to `_totalLocked` . now `_totalLocked` is 111 MYT + 99.9 MYT = **210.9 MYT**.

notice the `account.rawLocked` is each time recalculate the `lockedCollateral` + `toLock` using latest value per share which is now respectively: 99.9 MYT and the new debt requires 99.9 MYT to lock, for a new total of **199.8 MYT**.

this discrepancy itself already sufficient for a bug. but we can further check where the additional impact is:

```solidity
      function redeem(uint256 amount) external onlyTransmuter {
...
        uint256 old = _totalLocked;
        _totalLocked = totalOut > old ? 0 : old - totalOut;
        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);
```

when redeem happening, the `_collateralWeight` would be calculated using the `_totalLocked` and `totalOut` . now we know that `_totalLocked` is inflated, this means the `_collateralWeight` would be inaccurate and affect all users.

## Impact Details

`_totalLocked` would be inflated over time which later would severely affect how the `_collateralWeight` is calculated. potentially to an artificially low or "stunted" `_collateralWeight` over time.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L917-L925>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L632-L634>

## Proof of Concept

## Proof of Concept

there are multiple files that need to modified:

AlchemistV3.sol so we can see the `_totalLocked` value

```diff
diff --git a/src/AlchemistV3.sol b/src/AlchemistV3.sol
index 975dec0..ae05cec 100644
--- a/src/AlchemistV3.sol
+++ b/src/AlchemistV3.sol
@@ -129,6 +129,8 @@ contract AlchemistV3 is IAlchemistV3, Initializable {
     /// Locked collateral is the collateral that cannot be withdrawn due to LTV constraints
     uint256 private _totalLocked;
 
+    event Debug(uint256);
+
     /// @dev Total yield tokens deposited
     /// This is used to differentiate between tokens deposited into a CDP and balance of the contract
     uint256 private _mytSharesDeposited;
@@ -920,7 +927,10 @@ contract AlchemistV3 is IAlchemistV3, Initializable {
         if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();
 
         account.rawLocked = lockedCollateral + toLock;
+        emit Debug(_totalLocked);
         _totalLocked += toLock;
+        emit Debug(_totalLocked);
+        emit Debug(account.rawLocked);
         account.debt += amount;
         totalDebt += amount;
     }

```

VaultV2.sol, MockMYTVault.sol and so we can quickly simulate/mock that Vault accrue yield

```diff
diff --git a/lib/vault-v2/src/VaultV2.sol b/lib/vault-v2/src/VaultV2.sol
index 074093b..49a4b77 100644
--- a/lib/vault-v2/src/VaultV2.sol
+++ b/lib/vault-v2/src/VaultV2.sol
@@ -711,7 +711,7 @@ contract VaultV2 is IVaultV2 {
 
     /// @dev Returns corresponding shares (rounded down).
     /// @dev Takes into account performance and management fees.
-    function convertToShares(uint256 assets) external view returns (uint256) {
+    function convertToShares(uint256 assets) external virtual view returns (uint256) {
         return previewDeposit(assets);
     }

```

```diff
diff --git a/src/test/mocks/MockMYTVault.sol b/src/test/mocks/MockMYTVault.sol
index 16c5988..16eecf6 100644
--- a/src/test/mocks/MockMYTVault.sol
+++ b/src/test/mocks/MockMYTVault.sol
@@ -5,4 +5,13 @@ import {VaultV2} from "../../../lib/vault-v2/src/VaultV2.sol";
 
 contract MockMYTVault is VaultV2 {
     constructor(address admin, address collateral) VaultV2(admin, collateral) {}
+    // we override convertToShares to simplify PoC
+    uint256 convertRate;
+    function setMockConvertToShares(uint256 rate) public {
+        convertRate = rate;
+    }
+    function convertToShares(uint256 assets) external view override returns (uint256) {
+        if (convertRate == 0) return previewDeposit(assets);
+        return assets * 1e18 / convertRate;
+    }
 }
```

and lastly, the test itself:

```diff
diff --git a/src/test/AlchemistV3.t.sol b/src/test/AlchemistV3.t.sol
index d3efaad..10373b0 100644
--- a/src/test/AlchemistV3.t.sol
+++ b/src/test/AlchemistV3.t.sol
@@ -36,6 +36,7 @@ import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
 import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
 import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
 import {MockYieldToken} from "./mocks/MockYieldToken.sol";
+import {MockMYTVault} from "./mocks/MockMYTVault.sol";
 
 contract AlchemistV3Test is Test {
     // ----- [SETUP] Variables for setting up a minimal CDP -----
@@ -918,6 +919,47 @@ contract AlchemistV3Test is Test {
         assertEq(allowanceAfterTransfer, 0);
     }
 
+    function test_poc_amountLockedIssue() external {
+        uint256 amount = 1000e18;
+        uint256 ltv = 2e17;
+        vm.startPrank(address(0xbeef));
+        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
+        alchemist.deposit(amount, address(0xbeef), 0);
+
+        // a single position nft would have been minted to address(0xbeef)
+        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
+        alchemist.mint(tokenId, (amount * ltv) / FIXED_POINT_SCALAR, address(0xbeef));
+        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(address(0xbeef)), (amount * ltv) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
+        vm.stopPrank();
+
+        (uint256 deposited, uint256 userDebt,) = alchemist.getCDP(tokenId);
+
+        assertApproxEqAbs(deposited, amount, 1);
+        assertApproxEqAbs(userDebt, amount * ltv / FIXED_POINT_SCALAR, 1);
+
+        assertApproxEqAbs(
+            alchemist.getMaxBorrowable(tokenId),
+            (alchemist.convertYieldTokensToDebt(amount) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization()) - (amount * ltv) / FIXED_POINT_SCALAR,
+            1
+        );
+
+        assertApproxEqAbs(alchemist.getTotalUnderlyingValue(), alchemist.convertYieldTokensToUnderlying(amount), 1);
+
+        // we simulate that vault is accruing yield, so the shares would worth more
+        uint256 oldValue = alchemist.convertDebtTokensToYield(100e18);
+        MockMYTVault(address(vault)).setMockConvertToShares(1.001e18);
+        uint256 newValue = alchemist.convertDebtTokensToYield(100e18);
+        // assert that if you deposit before, you got more
+        // meaning that MYT is accruing yield
+        assertGt(oldValue, newValue);
+
+        // user add new debt after vault acrrued yield
+        vm.startPrank(address(0xbeef));
+        alchemist.mint(tokenId, (amount * ltv) / FIXED_POINT_SCALAR, address(0xbeef));
+        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(address(0xbeef)), (2 * amount * ltv) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
+        vm.stopPrank();
+    }
+
     function testMint_Variable_Amount(uint256 amount) external {
         amount = bound(amount, FIXED_POINT_SCALAR, accountFunds);
         uint256 ltv = 2e17;

```

we then call call `forge test --mt test_poc_amountLockedIssue -vvvv` to see our `Debug` event from second deposit:

```bash
    ├─ [84879] TransparentUpgradeableProxy::fallback(1, 200000000000000000000 [2e20], 0x000000000000000000000000000000000000bEEF)
    │   ├─ [84158] AlchemistV3::mint(1, 200000000000000000000 [2e20], 0x000000000000000000000000000000000000bEEF) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000bEEF
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000bEEF
    │   │   ├─ [2798] MockMYTVault::convertToShares(200000000000000000000 [2e20]) [staticcall]
    │   │   │   └─ ← [Return] 199800199800199800199 [1.998e20]
    │   │   ├─ [2798] MockMYTVault::convertToShares(200000000000000000000 [2e20]) [staticcall]
    │   │   │   └─ ← [Return] 199800199800199800199 [1.998e20]
    │   │   ├─ [2798] MockMYTVault::convertToShares(200000000000000000000 [2e20]) [staticcall]
    │   │   │   └─ ← [Return] 199800199800199800199 [1.998e20]
    │   │   ├─ emit Debug(: 222222222222222222200 [2.222e20])
    │   │   ├─ emit Debug(: 444222444222444222398 [4.442e20])
    │   │   ├─ emit Debug(: 444000444000444000396 [4.44e20])
```

the first debug is `_totalLocked` after first deposit where the asset per share is still 1:1

the second debug is `_totalLocked` after the second deposit when 1 share is about 1.001 asset worth

the third debug is `account.rawLocked` which shown that it is not the same as latest `_totalLocked` at second debug event.

this clearly prove that the `_totalLocked` over time would inflates.
