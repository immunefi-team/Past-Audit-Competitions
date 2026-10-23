# 90085 sc low the withdraw side of the share asset conversion has no zero output guard so redeem can burn a holder s shares while crediting nothing claimable

**Submitted on Aug 22nd 2026 at 22:03:45 UTC by @MangalJup for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90085
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`FirelightVault` guards exactly one direction of its share/asset rounding: `deposit()` refuses an input that would floor to zero shares, and the code says so in a comment. `redeem()` has no matching guard on the value it computes going the other way. Once the share price sits below 1 — the state every incident `payout()` leaves behind — a `redeem()` call for a small enough (but still nonzero) `shares` amount floors its computed `assets` to `0`. The call does not revert: the full `shares` are burned from the caller, `WithdrawRequest` fires as if a real withdrawal request was created, and only later, at `claimWithdraw`, does `NoWithdrawalAmount` reveal that nothing was ever credited.

## Vulnerability Details

The deposit path checks its own rounding output and explains why:

```solidity
// contracts/core/FirelightVault.sol:662-669
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    assets,
    true,
    Math.Rounding.Floor
);
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
   if (shares == 0) revert InvalidAmount();
```

`redeem()` checks only its *input*, never the value the same conversion produces:

```solidity
// contracts/core/FirelightVault.sol:731-741
if (shares == 0) revert InvalidAmount();   // guards the input only

(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    shares,
    false,
    Math.Rounding.Floor
);
   // nothing here checks whether `assets` also came out to zero

uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
```

`_requestWithdraw` then burns exactly `shares`, regardless of what `assets` turned out to be:

```solidity
// contracts/core/FirelightVault.sol:1005-1027
uint256 period = currentPeriod() + 1;
uint256 sharesWithdraw = _convertToSharesTotals(assets, withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor);
withdrawAssets[period] += assets;                        // += 0
withdrawShares[period] += sharesWithdraw;                 // += 0
withdrawSharesOf[period][receiver] += sharesWithdraw;     // += 0
pendingWithdrawAssets += assets;                          // += 0
...
   _update(owner, address(0), shares);                    // burns the full, nonzero shares regardless
emit WithdrawRequest(sender, receiver, owner, period, assets, shares);   // assets == 0, event still fires
```

The rounding that opens this window is `_convertToAssetsTotals` (`:1067-1074`): `shares.mulDiv(totalAssets()+1, totalSupply()+1, Floor)` with `_decimalsOffset() == 0`. That floors to `0` for any `shares < (totalSupply()+1)/(totalAssets()+1)` — i.e. for a growing range of nonzero `shares` values every time `totalAssets() < totalSupply()`, which is exactly the state left behind by any `payout()` (`:892-966`) that isn't a 0%-loss no-op.

## Impact Details

* The exact scenario `deposit()`'s own comment says the contract refuses to produce — "shares/assets in exchange for nothing" — happens anyway from the redeem side, with no revert to stop it.
* `redeem()`'s NatSpec (`:715`) promises its return value is "Amount of assets that will be received in the next period." It silently returns `0` instead of reverting, breaking that documented contract.
* `WithdrawRequest` fires on a request that credits nothing claimable. Anything reading that event as "a withdrawal is now pending" — an indexer, a subgraph, an ops dashboard — is wrong until the mismatch surfaces at `claimWithdraw`, which reverts `NoWithdrawalAmount` for a period that outwardly looked fine.
* No privileged role is needed to trigger it — any holder, or any spender the holder approved, can hit this the moment the share price is below 1, which is the vault's normal post-incident state.
* By construction of the flooring condition, the *fair value* actually destroyed per occurrence is under one asset-unit wei — this does not move meaningful value, which is why the impact category here is "fails to deliver promised returns," not fund loss.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L662-L669>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L731-L741>

## Proof of Concept

## Proof of Concept

One ordinary deposit, one ordinary incident payout to move the share price below 1, one `redeem()` call sized right at the dust boundary. No privileged trigger, no malicious actor. Save as `test/RedeemFloorToZero.t.sol` and run `forge test --match-contract RedeemFloorToZeroPoC -vv`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";

contract MockPeggedFXRP is ERC20 {
    constructor() ERC20("Mock Pegged FXRP", "mpFXRP") {}
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

contract RedeemFloorToZeroPoC is Test {
    address internal constant GOVERNOR = address(0x900D);
    address internal constant PAYOUT_SINK = address(0x51C0);  // allowlisted payout destination
    address internal constant LP_HOLDER = address(0x707A1);   // ordinary depositor, later redeems

    FirelightVault internal vault;
    MockPeggedFXRP internal asset;
    uint48 internal captureEpoch;

    event WithdrawRequest(
        address indexed sender,
        address indexed receiver,
        address indexed owner,
        uint256 period,
        uint256 assets,
        uint256 shares
    );

    function setUp() public {
        vm.warp(1_800_000_000);
        asset = new MockPeggedFXRP();

        FirelightVault impl = new FirelightVault();
        IFirelightVault.InitParams memory params = IFirelightVault.InitParams({
            defaultAdmin: GOVERNOR,
            limitUpdater: GOVERNOR,
            blocklister: GOVERNOR,
            pauser: GOVERNOR,
            periodConfigurationUpdater: GOVERNOR,
            rescuer: GOVERNOR,
            depositLimit: type(uint128).max,
            periodConfigurationDuration: 1 days
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(impl),
                    abi.encodeCall(
                        FirelightVault.initialize,
                        (IERC20(address(asset)), "Firelight Vault", "fvFXRP", abi.encode(params))
                    )
                )
            )
        );

        vm.startPrank(GOVERNOR);
        vault.grantRole(vault.PAYOUT_ROLE(), GOVERNOR);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), GOVERNOR);
        vault.addToPayoutAllowlist(PAYOUT_SINK);
        vm.stopPrank();

        captureEpoch = uint48(vm.getBlockTimestamp());
    }

    function _deposit(address who, uint256 amt) internal {
        asset.mint(who, amt);
        vm.startPrank(who);
        asset.approve(address(vault), amt);
        vault.deposit(amt, who);
        vm.stopPrank();
    }

    function test_redeemBurnsSharesForZeroCreditOnceSharePriceDropsBelowOne() public {
        // Step 1 -- ordinary LP deposit, share price exactly 1.
        _deposit(LP_HOLDER, 1_000_000e6);
        assertEq(vault.totalAssets(), vault.totalSupply(), "price should start at 1");

        // The deposit-side guard exists and works -- confirms the contract already
        // treats "shares/assets for nothing" as something worth reverting on.
        vm.prank(LP_HOLDER);
        vm.expectRevert(FirelightVault.InvalidAmount.selector);
        vault.deposit(0, LP_HOLDER);

        // Step 2 -- an ordinary incident payout drops totalAssets() below
        // totalSupply(), opening the window where redeem() can floor to zero.
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 payoutAmount = vault.totalAssets() - 100; // leave a sliver of assets alive
        vm.prank(GOVERNOR);
        vault.payout(PAYOUT_SINK, payoutAmount, captureEpoch);

        uint256 assetsLeft =
vault.totalAssets();
        uint256 sharesOutstanding = vault.totalSupply();
        assertLt(assetsLeft, sharesOutstanding, "dust window should be open post-payout");

        // Step 3 -- largest `shares` value that still floors to zero assets under
        // _convertToAssetsTotals' exact rounding rule.
        uint256 maxDustShares = (sharesOutstanding + 1) / (assetsLeft + 1);
        if (maxDustShares * (assetsLeft + 1) >= sharesOutstanding + 1) maxDustShares -= 1;
        assertGt(maxDustShares, 0, "dust window must be nonzero");

        uint256 sharesBefore = vault.balanceOf(LP_HOLDER);
        uint256 supplyBefore = vault.totalSupply();
        uint256 pendingBefore = vault.pendingWithdrawAssets();
        uint256 targetPeriod = vault.currentPeriod() + 1;

        // Step 4 -- redeem() succeeds (no revert), emits a success-shaped event with
        // assets == 0, and burns the caller's shares anyway.
        vm.expectEmit(true, true, true, true, address(vault));
        emit WithdrawRequest(LP_HOLDER, LP_HOLDER, LP_HOLDER, targetPeriod, 0, maxDustShares);
        vm.prank(LP_HOLDER);
        uint256 assetsReturned = vault.redeem(maxDustShares, LP_HOLDER, LP_HOLDER);

        assertEq(assetsReturned, 0, "redeem() should have returned exactly zero");
        assertEq(vault.balanceOf(LP_HOLDER), sharesBefore - maxDustShares, "shares must still be burned");
        assertEq(vault.totalSupply(), supplyBefore - maxDustShares, "supply must still drop");
        assertEq(vault.withdrawSharesOf(targetPeriod, LP_HOLDER), 0, "nothing was credited to claim");
        assertEq(vault.pendingWithdrawAssets(), pendingBefore, "pending pool untouched");

        console2.log("shares burned for zero credit  =", maxDustShares);
        console2.log("assets actually returned        =", assetsReturned);
        console2.log("claimable balance next period   =", vault.withdrawSharesOf(targetPeriod, LP_HOLDER));
    }
}
```

```
[PASS] test_redeemBurnsSharesForZeroCreditOnceSharePriceDropsBelowOne() (gas: 685268)
Logs:
  shares burned for zero credit  = 9900990099
  assets actually returned        = 0
  claimable balance next period   = 0

Suite result: ok. 1 passed; 0 failed; 0 skipped
```
