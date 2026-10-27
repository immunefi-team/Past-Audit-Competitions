# 88984 sc low missing zero value check in redeem silently burns shares for zero assets after totalassets reaches zero

**Submitted on Aug 16th 2026 at 08:08:34 UTC by @bytescannet for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88984
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`FirelightVault.redeem()` has no guard against a zero-value conversion result. `deposit()` explicitly reverts if the computed shares round to zero, but `redeem()` has no equivalent check for assets. Once `totalAssets()` reaches 0, a reachable state after a legitimate incident payout via `payout()`, any subsequent `redeem()` call burns the caller's full share balance and returns 0 assets, with no revert and no recovery path.

## Vulnerability Details

In `FirelightVault.sol`, `deposit()` enforces:

```solidity
if (shares == 0) revert InvalidAmount();
```

`redeem()` has no matching check:

```solidity
function redeem(uint256 shares, address receiver, address owner) public override ... returns (uint256) {
    if (shares == 0) revert InvalidAmount();
    (uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(shares, false, Math.Rounding.Floor);
    uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
    ...
```

`_convertToAssetsTotals` rounds down:

```solidity
assets = shares * (totalAssets + 1) / (totalSupply + offset)
```

When `totalAssets() == 0`, this floors to exactly 0 for every valid share amount, since shares can never exceed `totalSupply`.

`totalAssets() == 0` is reachable through normal, correctly-authorized protocol operation: `payout()` can transfer the vault's entire active balance to an approved incident claimant. This does not require any bug in `payout()` itself; it is the intended loss-absorption mechanism, and the impact list for this program includes "Protocol insolvency" as a recognized reachable state.

Once `totalAssets() == 0`, any depositor calling `redeem()` on their own shares:

* Has their full share balance burned (standard ERC20 burn via `_update`)
* Receives `assetsReturned == 0`
* Has a withdrawal request filed with `assets: 0` (visible in the emitted `WithdrawRequest` event)
* Cannot recover anything later: `claimWithdraw()` on that request reverts `NoWithdrawalAmount`, since the recorded claimable amount is 0

The shares are gone permanently. If the vault later recovers value (new deposits, reward forwarding via `VaultRewardDistributor`), the affected user has no claim on that recovery, since their shares no longer exist.

This is not an attacker-vs-victim bug. `redeem()` requires either `msg.sender == owner` or a prior ERC20 allowance from owner (via `_spendAllowance`), so no unprivileged third party can force this against another user's shares. The bug fires against whoever calls `redeem()` on their own position during the zero-asset window, including a user acting in complete good faith with no attacker involved.

## Impact

A user who redeems during a `totalAssets() == 0` window loses their entire remaining claim on the vault for zero returned value, with no revert to warn them and no way to recover afterward. This is a real, non-contrived code path: it is reached by the protocol's own designed loss-absorption mechanism (`payout()`), not by any external manipulation.

## Recommendation

Add a symmetric zero-value guard to `redeem()`, mirroring the existing check in `deposit()`:

```solidity
if (assets == 0) revert InvalidAmount();
```

Place it immediately after the `_previewTotals()` call and before `_requestWithdraw()` is invoked.

## Proof of Concept

### Environment

* Foundry (forge 1.7.1), solc 0.8.28, via-ir
* Repo: audit-comp-firelight, branch `v1_audit_ready`
* OpenZeppelin @5.2.0 (contracts) / @5.4.0 (contracts-upgradeable), matching the versions pinned in `package.json`

### Step-by-step

{% stepper %}
{% step %}

## Deploy and initialize the vault

Deploy `FirelightVault` behind an `ERC1967Proxy` and initialize it with a mock ERC20 asset, admin address holding all init roles.
{% endstep %}

{% step %}

## Configure payout permissions

Grant `PAYOUT_ROLE` and `PAYOUT_ALLOWLIST_ROLE` to admin, and allowlist a payout receiver.

This isolates the vault-side accounting bug from `IncidentManager`'s approval workflow; the bug lives entirely in `redeem()`'s accounting after a payout lands, not in how that payout was authorized upstream.
{% endstep %}

{% step %}

## Deposit assets

`depositor` deposits `1000e18` of the asset and receives `1000e18` shares (1:1 first deposit).

`totalAssets() == 1000e18`.
{% endstep %}

{% step %}

## Confirm normal redemption behavior

`previewRedeem(1000e18)` returns `1000e18`, confirming correct pre-incident behavior.
{% endstep %}

{% step %}

## Drain the active balance through a legitimate payout

Admin calls `vault.payout(receiver, 1000e18, captureTimestamp)`, draining the vault's full active balance.

This is a legitimate, correctly-authorized call representing an approved incident claim, not a bug in `payout()` itself, and is the vault's designed loss-absorption mechanism.
{% endstep %}

{% step %}

## Verify zero assets while shares remain

`totalAssets()` is now 0. `depositor` still holds the full `1000e18` shares, since `payout()` moves assets only, not shares.
{% endstep %}

{% step %}

## Observe the zero-value conversion

`previewRedeem(1000e18)` now returns 0.

The bug is already visible here, in a read-only call, before any state change.
{% endstep %}

{% step %}

## Redeem shares for zero assets

`depositor` calls `redeem(1000e18, depositor, depositor)`.

* `Transfer(depositor -> 0x0, 1000e18)` fires: the full share balance is burned.
* `WithdrawRequest(..., period: 1, assets: 0, shares: 1000e18)` fires: the request itself is logged with `assets: 0`.
* `redeem()` returns 0.
  {% endstep %}

{% step %}

## Confirm the shares are gone

`balanceOf(depositor)` is now 0. Shares are gone.
{% endstep %}

{% step %}

## Confirm there is no recovery path

Time is warped past that request's claim window. `claimWithdraw(1)` is called and reverts `NoWithdrawalAmount(1)`, confirming there is no later recovery path for the burned position.
{% endstep %}
{% endstepper %}

### Result (actual forge test run, not a manual trace)

```
[PASS] test_RedeemBurnsSharesForZeroAssetsAfterFullPayout() (gas: 552749)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Key trace excerpt:

```
FirelightVault::payout(receiver, 1000e18, ...) -> emit PayoutExecuted(paidAmount: 1000e18)
FirelightVault::totalAssets() -> 0
FirelightVault::redeem(1000e18, depositor, depositor)
  -> emit Transfer(from: depositor, to: 0x0, amount: 1000e18)
  -> emit WithdrawRequest(..., period: 1, assets: 0, shares: 1000e18)
  -> returns 0
FirelightVault::claimWithdraw(1) -> Revert NoWithdrawalAmount(1)
```

### Full test file

Foundry, self-contained. Run with:

```bash
forge test --match-contract RedeemZeroAssetsPoC -vvvv
```

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../../core/FirelightVault.sol";

contract PoCMockAsset is ERC20 {
    constructor() ERC20("Mock Asset", "MOCK") {
        _mint(msg.sender, 1_000_000 ether);
    }
}

contract RedeemZeroAssetsPoC is Test {
    FirelightVault public vault;
    PoCMockAsset public asset;

    address admin = address(0xA11CE);
    address depositor = address(0xB0B);
    address receiver = address(0xD00D);

    uint256 constant DEPOSIT_AMOUNT = 1_000 ether;

    function setUp() public {
        asset = new PoCMockAsset();
        asset.transfer(depositor, DEPOSIT_AMOUNT);

        FirelightVault implementation = new FirelightVault();

        FirelightVault.InitParams memory initParams = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: 1_000_000 ether,
            periodConfigurationDuration: 1 days
        });

        bytes memory initData = abi.encodeWithSelector(
            FirelightVault.initialize.selector,
            IERC20(address(asset)),
            "Firelight Vault",
            "fvUSD",
            abi.encode(initParams)
        );

        ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), initData);
        vault = FirelightVault(address(proxy));

        vm.startPrank(admin);
        vault.grantRole(vault.PAYOUT_ROLE(), admin);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), admin);
        vault.addToPayoutAllowlist(receiver);
        vm.stopPrank();

        vm.prank(depositor);
        asset.approve(address(vault), DEPOSIT_AMOUNT);
    }

    function test_RedeemBurnsSharesForZeroAssetsAfterFullPayout() public {
        vm.prank(depositor);
        uint256 shares = vault.deposit(DEPOSIT_AMOUNT, depositor);
        assertEq(shares, DEPOSIT_AMOUNT, "expected 1:1 first deposit");
        assertEq(vault.totalAssets(), DEPOSIT_AMOUNT, "vault should hold full deposit");

        uint256 previewBefore = vault.previewRedeem(shares);
        assertEq(previewBefore, DEPOSIT_AMOUNT, "pre-incident redeem should return full value");

        vm.prank(admin);
        uint256 paid = vault.payout(receiver, DEPOSIT_AMOUNT, uint48(block.timestamp));
        assertEq(paid, DEPOSIT_AMOUNT, "payout should drain the full active balance");
        assertEq(vault.totalAssets(), 0, "vault totalAssets should now be zero");
        assertEq(asset.balanceOf(receiver), DEPOSIT_AMOUNT, "payout receiver got the funds");
        assertEq(vault.balanceOf(depositor), DEPOSIT_AMOUNT, "depositor still holds all shares post-payout");

        uint256 previewAfter = vault.previewRedeem(shares);
        assertEq(previewAfter, 0, "preview now shows zero, as expected given the drain");

        vm.prank(depositor);
        uint256 assetsReturned = vault.redeem(shares, depositor, depositor);

        assertEq(assetsReturned, 0, "F-01: redeem returned zero assets");
        assertEq(vault.balanceOf(depositor), 0, "F-01: shares were fully burned anyway");

        uint256 requestPeriod = vault.currentPeriod() + 1;
        assertEq(requestPeriod, 1, "sanity: request should be filed for period 1");

        vm.warp(block.timestamp + 2 days);
        assertTrue(vault.currentPeriod() > requestPeriod, "sanity: must be past the request period");

        vm.prank(depositor);
        vm.expectRevert(
            abi.encodeWithSelector(FirelightVault.NoWithdrawalAmount.selector, requestPeriod)
        );
        vault.claimWithdraw(requestPeriod);
    }
}
```
