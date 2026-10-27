# 89777 sc high valid target period withdrawal becomes unclaimable after rescue and unblock&#x20;

**Submitted on Aug 21st 2026 at 04:02:25 UTC by @Diavol0 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89777
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

### Brief/Intro

The blocklist -> rescue -> unblock sequence is a supported account-management lifecycle that can arise during routine compliance operations; it does not require the affected user to control or influence any role holder. Even when every privileged party acts within its documented permissions, uses valid parameters, and makes no malicious or intentionally incorrect business decision, a rescued withdrawal can leave a stale terminal marker behind. After a pending withdrawal is rescued for a blocklisted account, the vault marks that account as claimed for the period. The documented unblock path then restores the account's normal vault access, including `redeem()` and `withdraw()`, but `_requestWithdraw()` does not clear or version the old marker. If the restored account makes no further request, the stale state is dormant; if it deposits fresh assets after unblocking and makes one ordinary request targeting the same period, the new entitlement is recorded and counted in `pendingWithdrawAssets`, yet both `claimWithdraw()` and a later rescue revert with `AlreadyClaimedPeriod`, leaving that principal inaccessible under the deployed implementation. No incident creation, assessment, approval, payout, allocation, premium, or oracle action is involved. The failure is therefore produced by the later permissionless request path, not by privilege escalation, role misuse, malformed input, or an incorrect incident/assessment/payout decision.

### Vulnerability Details

`isWithdrawClaimed[period][account]` is used as a terminal gate by both consumers of a pending withdrawal:

```solidity
// FirelightVault.claimWithdraw
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

assets = _convertToAssetsTotals(
    withdrawSharesOf[period][sender], withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor
);
if (assets == 0) revert NoWithdrawalAmount(period);

pendingWithdrawAssets -= assets;
isWithdrawClaimed[period][sender] = true;
IERC20(asset()).safeTransfer(sender, assets);
```

The rescue path also terminalizes the source account, even though the shares are moved to the rescue beneficiary:

```solidity
// FirelightVault.rescueWithdrawFromBlocklisted
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

However, the withdrawal-request path accepts another entitlement for the same `(period, receiver)` without checking or updating that terminal state. In particular, `redeem()` and `withdraw()` only check blocklist/pause status before calling this path; a successful request burns the user's shares and records a new liability:

```solidity
// FirelightVault._requestWithdraw
uint256 period = currentPeriod() + 1;
uint256 sharesWithdraw = _convertToSharesTotals(
    assets, withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor
);
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
```

The resulting invariant failure is direct: once `isWithdrawClaimed[period][account]` is `true`, any positive `withdrawSharesOf[period][account]` must either be rejected at creation or have independent claim state. The implementation instead accepts and reserves the positive balance while both consumers unconditionally reject it.

`removeFromBlocklist()` is explicitly documented as removing an account from the blocklist and only sets `isBlocklisted[account] = false`; it neither forbids future withdrawals nor resets the account-period marker. This makes the unblock operation a supported restoration of normal vault access while leaving the stale terminal state behind. The vault's `redeem()` and `withdraw()` functions are likewise documented to create delayed withdrawal requests, so the later request is a normal user action rather than an invalid edge-case call.

The finding does not assume that adding and later removing an account was an incorrect business decision. Both operations are separately exposed, documented state transitions, and `removeFromBlocklist()` has no guard for an account whose withdrawal was previously rescued. If the intended policy were that rescue permanently forbids a restored account from creating a request for that target period, that policy must be enforced by the contract before `_requestWithdraw()` burns shares or records `pendingWithdrawAssets`; treating the supported unblock call itself as an error leaves the accepted liability/state contradiction unaddressed.

The following normal lifecycle is sufficient to create an unclaimable liability. It uses no malformed input and no authority outside the documented account-management and rescue functions:

1. While period `0` is current, a user redeems `100 FXRP`. The request is assigned to period `1`.
2. The account is added to the blocklist, and the pending period-1 withdrawal is rescued to a non-blocklisted treasury. The old withdrawal is now represented by `withdrawSharesOf[1][treasury]`; `isWithdrawClaimed[1][user]` is set to `true` as the terminal marker for the source account.
3. The account is removed from the blocklist through the normal unblock transition. Before the current period changes, the user deposits a fresh `50 FXRP` and then redeems those newly deposited shares to itself (`receiver == owner == user`). Since `_requestWithdraw()` sets `period = currentPeriod() + 1`, this request targets period `1`, increments `withdrawSharesOf[1][user]`, and leaves the old `true` marker unchanged.
4. After period `1` ends, the treasury successfully claims its rescued `100 FXRP`. The user's new `50 FXRP` entitlement cannot be claimed because `claimWithdraw(1)` checks the stale marker before calculating the amount.
5. Adding the user to the blocklist again does not provide a recovery path. `rescueWithdrawFromBlocklisted(user, treasury, [1])` reaches the same marker check and reverts before moving the new shares.

At the time of both redemptions, `currentPeriod() == 0`, so both requests target withdrawal period `1` through `currentPeriod() + 1`. The fresh deposit and redemption only need to occur before the current period rolls over; they do not require a period-boundary timestamp, same-block ordering, validator timestamp control, or transaction race. A request to a fresh receiver would avoid creating another bad slot, but cannot move or claim the already-written `withdrawSharesOf[1][user]` balance. The self-directed request (`receiver == owner == user`) is the canonical ERC4626 withdrawal form and is not prohibited by the vault.

The second `redeem()` is a successful standard withdrawal request: it passes all access and blocklist checks, burns the user's shares, updates the pending-liability accounting, and emits the normal withdrawal event. The contract gives no warning or revert that the selected receiver-period slot is already terminalized; the failure appears only when the protocol later consumes the accepted request.

This is a conditional state-machine failure, not a claim that every unblock operation is harmful. If the restored account performs no further withdrawal before period `0` ends, the stale marker has no visible effect. If that account performs the ordinary `redeem()` or `withdraw()` operation that the vault advertises, the request targets period `1` and is accepted into the same account-period slot that was already terminalized by the rescue. The contract therefore has two outcomes for the same valid restoration flow: no new request, no issue; one normal target-period request, an accepted but unclaimable principal balance. A correct implementation must reject the second request before burning/recording assets or give it independent claim state.

The faulty transition is the permissionless request in step 3: the contract accepts and accounts for a new entitlement in a state that its own terminal marker says is already consumed. `rescueWithdrawFromBlocklisted()` establishes the terminal state, and both `claimWithdraw()` and a later rescue enforce it unconditionally for the same `(period, account)` slot. Once that state exists, `_requestWithdraw()` must either reject a new request for that slot before accounting for assets or create independent claim state for the new request. The current implementation does neither: it records the entitlement while leaving every available consumer path disabled.

The blocklist, rescue, and unblock calls only establish a reachable pre-state through documented functions. They do not require the affected user to control, compromise, or influence a role holder, and no role holder needs to choose an invalid value. The affected account need not be malicious; it is simply the account whose valid pending withdrawal was rescued and later restored. Once that pre-state exists, the failure is triggered by the restored account's permissionless `redeem()` call: `_requestWithdraw()` accepts a new entitlement even though the account-period marker is already terminalized. The privileged calls are therefore only a prerequisite state transition; the exploitable defect is the later ordinary user operation that creates an entitlement neither consumer path can process. After that request is accepted, no ordinary privileged or permissionless function can clear the marker or recover the already-written entitlement.

No incident assessor, assessment approver, cover-order operator, or payout coordinator is used, and no incident timestamp, assessment, allocation, premium, or payout amount is manipulated. The only privileged calls are the documented `BLOCKLIST_ROLE` and `RESCUER_ROLE` account-management/rescue functions, whose only effect is to establish the pre-state. The loss follows deterministically from the user's ordinary post-unblock withdrawal, even when every privileged role follows its intended policy. If same-target-period requests after rescue are unsupported, `_requestWithdraw()` must reject them before burning/recording assets; silently accepting and reserving the assets creates the unclaimable liability.

Suggested remediation: either reject a new request for an account-period that has already been terminalized, or replace the boolean with request-specific claim/rescue identifiers. Merely clearing the boolean on every new request would be unsafe for ordinary claims because `withdrawSharesOf` is cumulative and previously claimed shares must not become claimable again.

### Impact Details

The selected in-scope impact is **High - Temporary freezing of funds for at least 24 hour**. The newly accepted withdrawal amount is principal, not yield, and is inaccessible through every ordinary claim or rescue path:

* `withdrawSharesOf[period][user]` remains nonzero, but `claimWithdraw()` always reverts with `AlreadyClaimedPeriod`;
* a subsequent rescue cannot move it because `rescueWithdrawFromBlocklisted()` checks the same marker first;
* `pendingWithdrawAssets` continues to reserve the amount while `totalAssets()` subtracts it, so the frozen amount is excluded from the vault's active assets;
* there is no expiry, reset, or permissionless recovery function for the `(period, user)` marker.

In the runnable PoC, `50 FXRP` is accepted into period `1`, remains in the vault, and is unclaimable after the prior `100 FXRP` rescue. The amount scales with the user's ordinary withdrawal and can be the user's entire remaining withdrawal balance. The failure is reached when period `1` becomes claimable; the contract is not merely delaying payment because the only period-specific claim and rescue paths then revert deterministically. Recovery requires a governance-controlled implementation upgrade or storage migration rather than an ordinary claim or rescue operation available to the affected user. The broken entitlement therefore remains frozen beyond the normal claim window, whose configured minimum period duration is one day, until that intervention is deployed.

The initial `100 FXRP` is not part of the loss: it is successfully transferred to the rescue beneficiary. The frozen amount is only the fresh `50 FXRP` that the contract accepted after the account was restored.

### References

* [FirelightVault.sol `claimWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L753-L777) - terminal claim marker is checked before the withdrawal is read.
* [FirelightVault.sol `rescueWithdrawFromBlocklisted`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L808-L834) - rescue marks the source account as claimed.
* [FirelightVault.sol blocklist lifecycle](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L544-L560) - adding and removing a blocklist entry are separate documented state transitions.
* [FirelightVault.sol `_requestWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L954-L982) - a same-target-period request is appended without reconciling the marker.
* [IFirelightVault.sol delayed-withdrawal interface](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/interfaces/IFirelightVault.sol#L14-L18) - shares are burned on request and assets are later claimed by period.
* [FirelightVault.sol `totalAssets`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L424-L430) - pending withdrawals are excluded from active vault assets.

## Proof of Concept

Save the following as `test/V05Standalone.t.sol` in the repository's existing Foundry harness (`.audit-simao-pIQTSd/poc`) and run:

```bash
forge test --match-path test/V05Standalone.t.sol -vv
```

Expected result:

```
[PASS] test_rescueThenUnblockSamePeriodRequestRemainsUnclaimable()
1 passed; 0 failed
```

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {FirelightVault} from "firelight/core/FirelightVault.sol";

contract V05Token is ERC20 {
    constructor() ERC20("Fixture FXRP", "fFXRP") {}

    function decimals() public pure override returns (uint8) {
        return 6;
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract V05Standalone is Test {
    uint48 private constant START = 1_700_000_000;
    uint256 private constant UNIT = 1e6;
    address private constant ADMIN = address(0xADAD);
    address private constant BLOCKLISTER = address(0xB10C);
    address private constant RESCUER = address(0x5E5C);
    address private constant USER = address(0xA11CE);
    address private constant TREASURY = address(0x7EA5);

    V05Token private asset;
    FirelightVault private vault;

    function setUp() public {
        vm.warp(START);
        asset = new V05Token();

        FirelightVault implementation = new FirelightVault();
        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: ADMIN,
            limitUpdater: address(0),
            blocklister: BLOCKLISTER,
            pauser: address(0),
            periodConfigurationUpdater: address(0),
            rescuer: RESCUER,
            depositLimit: 10_000 * UNIT,
            periodConfigurationDuration: 1 days
        });

        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(implementation),
                    abi.encodeCall(
                        FirelightVault.initialize,
                        (asset, "Fixture Vault", "fVAULT", abi.encode(params))
                    )
                )
            )
        );

        asset.mint(USER, 1_000 * UNIT);
        vm.startPrank(USER);
        asset.approve(address(vault), type(uint256).max);
        vault.deposit(1_000 * UNIT, USER);
        vm.stopPrank();
    }

    function test_rescueThenUnblockSamePeriodRequestRemainsUnclaimable() public {
        assertEq(vault.currentPeriod(), 0);
        vm.prank(USER);
        vault.redeem(100 * UNIT, USER, USER);
        vm.warp(START + 1 hours);
        assertEq(vault.currentPeriod(), 0);

        vm.prank(BLOCKLISTER);
        vault.addToBlocklist(USER);
        vm.warp(START + 2 hours);
        assertEq(vault.currentPeriod(), 0);
        uint256[] memory periods = new uint256[](1);
        periods[0] = 1;
        vm.prank(RESCUER);
        vault.rescueWithdrawFromBlocklisted(USER, TREASURY, periods);
        assertTrue(vault.isWithdrawClaimed(1, USER));
        assertEq(vault.withdrawSharesOf(1, USER), 0);
        vm.warp(START + 3 hours);
        assertEq(vault.currentPeriod(), 0);

        vm.prank(BLOCKLISTER);
        vault.removeFromBlocklist(USER);
        assertFalse(vault.isBlocklisted(USER));
        vm.warp(START + 4 hours);
        assertEq(vault.currentPeriod(), 0);

        // Supply new user funds after unblock, then request a withdrawal for those
        // newly deposited shares. The old terminal marker still applies to period 1.
        // Fixture-only mint simulates the user acquiring fresh external assets;
        // it does not call or modify any vault role.
        asset.mint(USER, 50 * UNIT);
        vm.prank(USER);
        vault.deposit(50 * UNIT, USER);
        assertEq(vault.balanceOf(USER), 950 * UNIT);

        vm.prank(USER);
        vault.redeem(50 * UNIT, USER, USER);
        assertEq(vault.withdrawalsOf(1, USER), 50 * UNIT);
        assertEq(vault.balanceOf(USER), 900 * UNIT);
        assertEq(vault.pendingWithdrawAssets(), 150 * UNIT);
        assertTrue(vault.isWithdrawClaimed(1, USER));

        vm.warp(START + 2 days);
        vm.prank(TREASURY);
        assertEq(vault.claimWithdraw(1), 100 * UNIT);
        assertEq(asset.balanceOf(address(vault)), 950 * UNIT);

        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, 1));
        vm.prank(USER);
        vault.claimWithdraw(1);

        vm.prank(BLOCKLISTER);
        vault.addToBlocklist(USER);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, 1));
        vm.prank(RESCUER);
        vault.rescueWithdrawFromBlocklisted(USER, TREASURY, periods);
        assertEq(vault.pendingWithdrawAssets(), 50 * UNIT);
    }
}
```
