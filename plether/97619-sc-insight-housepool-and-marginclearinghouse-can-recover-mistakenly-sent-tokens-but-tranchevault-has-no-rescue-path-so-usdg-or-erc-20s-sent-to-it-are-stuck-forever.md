# \#97619 [SC-Insight] `HousePool` and `MarginClearinghouse` can recover mistakenly sent tokens, but `TrancheVault` has no rescue path, so USDG or ERC-20s sent to it are stuck forever

**Submitted on Nov 2nd 2026 at 14:39:37 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97619
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

### Brief/Intro

LPs interact with the Senior and Junior `TrancheVault` addresses, and both vaults hold USDG. If someone sends USDG straight to a vault instead of calling `requestDeposit`, or sends any other ERC-20 to it, nobody can get those tokens back, including the owner and `HousePool`. The team already fixed this class for `MarginClearinghouse` (Cecuro LOW-6, `rescueToken`) and gave `HousePool` an `accountExcess` / `sweepExcess` path. The team accepted the clearinghouse USDG limitation because it "has no aggregate credited-balance counter". That reason does not apply to `TrancheVault`, which tracks its required USDG escrow exactly. The vault was left out of the fix.

### Vulnerability Details

`TrancheVault` knows exactly how much USDG it must hold:

```solidity
// TrancheVault.sol:133-136
@>  uint256 public pendingDepositEscrowAssets;
    uint256 public pendingRedeemEscrowShares;
    uint256 public depositClaimEscrowShares;
@>  uint256 public withdrawalEscrowAssets;
```

Every outbound asset transfer is bounded by these counters. No function moves `balanceOf(this) - (pendingDepositEscrowAssets + withdrawalEscrowAssets)`, and no function moves any token other than `asset()`:

- `TrancheVault.sol:872`: cancel refund, `assets = position.assets`
- `TrancheVault.sol:1211`: redeem claim, bounded by `withdrawalEscrowAssets`
- `TrancheVault.sol:1258`: dust to pool, `fundedAssets - claimedAssets`
- `TrancheVault.sol:1473`: epoch finalization, `epoch.assets`

The vault does not inherit `Ownable`. Its only privileged hooks are `onlyPool` settlement functions and the `onlyPoolOwner` maintenance-fee setters. There is no `rescueToken`, `sweepExcess` or `accountExcess` equivalent. The only "sweep" in the file is the internal `_sweepRedeemRefundDust` (`:1263`), which moves epoch dust to the pool and cannot touch unsolicited balances.

The sibling contracts have the recovery path:

```solidity
// HousePool.sol:585-596
@>  function sweepExcess(address recipient, uint256 amount) external onlyOwner {
        ...
        if (amount > excessAssets()) revert HousePool__ExcessAmountTooHigh();
        settlementToken.safeTransfer(recipient, amount);

// MarginClearinghouse.sol:324-337
@>  function rescueToken(address token, address recipient) external onlyOwner nonReentrant {
        if (token == settlementAsset) revert MarginClearinghouse__CannotRescueSettlementAsset();
```

The docs promise recovery for those two contracts and justify the clearinghouse USDG limitation with a reason that does not hold for the vault:

> "rescue accidentally transferred non-settlement tokens from the Clearinghouse" (`packages/perps/SECURITY.md:246-247`)
>
> "The clearinghouse has no aggregate credited-balance counter with which to distinguish recoverable surplus from funds backing account obligations." (`packages/perps/SECURITY.md:1453-1455`)

For `TrancheVault` the surplus has an exact, safe formula: `balance - (pendingDepositEscrowAssets + withdrawalEscrowAssets)`.

### Impact Details

No protocol or LP funds are at risk and the accounting is unaffected: the surplus is not part of `totalAssets()` (read from `HousePool`) and the settlement monitor only flags `actual < required` (`SettlementMonitorLensSidecar.sol:826-834`). The loss falls only on the sender.

The vault is a likely target for this mistake: LP UIs and ERC-7540 integrations show it as "the vault", and it is the `approve` spender for deposits. A user who calls `USDG.transfer(vault, x)` instead of `requestDeposit(x, ...)` loses `x` permanently. The same applies to any airdropped or mistakenly sent ERC-20. Governance cannot help even though the exact amount is computable on-chain. PoC: 1,000 USDG (`1_000e6`) and 500 foreign tokens stranded.

### Mitigation

Add a pool-owner-gated function that can never touch escrow:

```diff
+    function recoverUnaccounted(address token, address recipient) external onlyPoolOwner {
+        if (recipient == address(0)) revert TrancheVault__ZeroAddress();
+        if (token == address(this)) revert TrancheVault__EscrowInvariant(); // never move escrowed shares
+        uint256 bal = IERC20(token).balanceOf(address(this));
+        uint256 amount = token == asset()
+            ? bal - (pendingDepositEscrowAssets + withdrawalEscrowAssets)
+            : bal;
+        IERC20(token).safeTransfer(recipient, amount);
+    }
```

Alternative: route surplus USDG to `address(POOL)`, where the existing `accountExcess` / `sweepExcess` flow handles it.

### References

- `packages/perps/src/TrancheVault.sol:133-136`: escrow counters
- `packages/perps/src/TrancheVault.sol:872, 1211, 1258, 1473`: every asset transfer, all counter-bounded
- `packages/perps/src/TrancheVault.sol:237-249`: only `onlyPool` / `onlyPoolOwner` hooks
- `packages/perps/src/HousePool.sol:563-566, 571, 585-596`: `excessAssets`, `accountExcess`, `sweepExcess`
- `packages/perps/src/MarginClearinghouse.sol:324-337`: `rescueToken`
- `packages/perps/SECURITY.md:246-247, 1449-1455`: recovery promise and counter rationale
- Prior report: Cecuro LOW-6 (clearinghouse only, addressed for non-settlement tokens)

## Proof of Concept

## Proof of Concept

**Conditions under which this insight is valuable**
- A user sends USDG or any ERC-20 directly to a `TrancheVault` instead of using `requestDeposit`. 
- The LOW-6 fix covers `MarginClearinghouse` and `HousePool` has `sweepExcess`; the vault has neither, so the same mistake is unrecoverable only here.

Save as `packages/perps/test/insights_integration/TrancheVaultNoRescue.t.sol` and run `forge test --root packages/perps --match-path test/insights_integration/TrancheVaultNoRescue.t.sol -vvv`. It reuses the repo's `BasePerpTest` fixture (full stack: engine, HousePool, Senior/Junior vaults, oracle, router).

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {MockSettlementToken} from "../utils/MockSettlementToken.sol";

contract TrancheVaultNoRescuePoC is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);

    function test_TrancheVaultStrandsUnsolicitedTokens() public {
        uint256 mistaken = 1_000e6;

        // 1. ALICE sends USDG straight to the junior vault instead of requestDeposit.
        settlementToken.mint(ALICE, mistaken);
        vm.prank(ALICE);
        settlementToken.transfer(address(juniorVault), mistaken);

        // 2. A foreign token is also sent to the vault by mistake.
        MockSettlementToken foreign = new MockSettlementToken();
        foreign.mint(address(juniorVault), 500e6);

        uint256 escrow = juniorVault.pendingDepositEscrowAssets() + juniorVault.withdrawalEscrowAssets();
        uint256 surplus = settlementToken.balanceOf(address(juniorVault)) - escrow;
        assertEq(surplus, mistaken, "vault counters isolate the surplus exactly");

        // 3. A full ordinary LP cycle leaves the surplus untouched.
        _fundJunior(BOB, 10_000e6);
        escrow = juniorVault.pendingDepositEscrowAssets() + juniorVault.withdrawalEscrowAssets();
        assertEq(settlementToken.balanceOf(address(juniorVault)) - escrow, mistaken, "surplus never absorbed");

        // 4. No recovery entrypoint exists on the vault.
        (bool ok1,) = address(juniorVault).call(
            abi.encodeWithSignature("rescueToken(address,address)", address(foreign), address(this))
        );
        (bool ok2,) = address(juniorVault).call(
            abi.encodeWithSignature("sweepExcess(address,uint256)", address(this), mistaken)
        );
        assertFalse(ok1, "no rescueToken on TrancheVault");
        assertFalse(ok2, "no sweepExcess on TrancheVault");
        assertEq(foreign.balanceOf(address(juniorVault)), 500e6, "foreign token stuck");

        // 5. Contrast: the same mistakes are recoverable on HousePool and MarginClearinghouse.
        settlementToken.mint(ALICE, mistaken);
        vm.prank(ALICE);
        settlementToken.transfer(address(pool), mistaken);
        assertEq(pool.excessAssets(), mistaken);
        pool.sweepExcess(ALICE, mistaken);
        assertEq(settlementToken.balanceOf(ALICE), mistaken, "HousePool returns unsolicited USDG");

        foreign.mint(address(clearinghouse), 500e6);
        clearinghouse.rescueToken(address(foreign), ALICE);
        assertEq(foreign.balanceOf(ALICE), 500e6, "Clearinghouse returns foreign tokens");

        emit log_named_uint("USDG stranded in TrancheVault (6 dp)", surplus);
        emit log_named_uint("foreign token stranded in TrancheVault", foreign.balanceOf(address(juniorVault)));
    }

}
```

Result (from the insight file, `forge test --root packages/perps --match-path test/insights_integration/TrancheVaultNoRescue.t.sol -vvv`):

```
Ran 1 test for test/insights_integration/TrancheVaultNoRescue.t.sol:TrancheVaultNoRescuePoC
[PASS] test_TrancheVaultStrandsUnsolicitedTokens() (gas: 1434926)
Logs:
  USDG stranded in TrancheVault (6 dp): 1000000000
  foreign token stranded in TrancheVault: 500000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

