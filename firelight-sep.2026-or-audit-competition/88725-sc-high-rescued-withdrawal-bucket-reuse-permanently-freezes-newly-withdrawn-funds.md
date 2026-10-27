# 88725 sc high rescued withdrawal bucket reuse permanently freezes newly withdrawn funds

## #88725 \[SC-High] Rescued withdrawal bucket reuse permanently freezes newly withdrawn funds

**Submitted on Aug 14th 2026 at 17:45:54 UTC by @v\_c0d35 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88725
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

### Description

#### Brief/Intro

In the public `v1_audit_ready` source at commit **`42f9ea5e43d88b35197b901acc2a8c24b314fa62`**, `FirelightVault` permanently marks Alice's period bucket claimed during rescue but later accepts and accounts for a new withdrawal into that same bucket. Once Alice is permitted to use the vault again, every withdrawal the vault accepts and burns shares for must remain claimable or rescuable; instead, Alice's mature claim and a later authorized rescue both reject the terminal marker, permanently freezing the newly accepted withdrawal.

#### Vulnerability Details

The critical invariant is that a terminal claimed flag must never coexist with a newly accepted entitlement under the same `(period, receiver)` key. Blocklisting an account, rescuing its pending withdrawal to a beneficiary, and later removing the account from the blocklist are supported compliance operations. When those operations complete before the current period rolls over, `currentPeriod() + 1` deterministically reuses the same future withdrawal bucket.

The reproducible sequence follows the contract's ordinary role boundaries:

{% stepper %}
{% step %}

### Alice requests a withdrawal

Alice deposits 10,000 six-decimal fXRP and requests a withdrawal to herself for future period `P`.
{% endstep %}

{% step %}

### Compliance blocklists Alice

Compliance blocklists Alice.
{% endstep %}

{% step %}

### An authorized rescuer moves Alice's pending withdrawal

The authorized rescuer moves Alice's pending withdrawal for `P` to non-blocklisted Bob. The rescue sets `isWithdrawClaimed[P][Alice] = true`.
{% endstep %}

{% step %}

### Compliance removes Alice from the blocklist

Compliance removes Alice from the blocklist before the current period changes.
{% endstep %}

{% step %}

### Alice requests a new withdrawal

Alice deposits another 10,000 fXRP and requests a new withdrawal to herself. Since `currentPeriod() + 1` still equals `P`, the vault appends the new entitlement to Alice's already-claimed bucket and burns her shares.
{% endstep %}

{% step %}

### Claims and rescue both fail

After `P` matures, Bob successfully claims the original rescued withdrawal. Alice's claim reverts with `AlreadyClaimedPeriod(P)`.

Compliance blocklists Alice again, but an authorized re-rescue also reverts with `AlreadyClaimedPeriod(P)`.
{% endstep %}
{% endstepper %}

The sequence requires compliance to remove Alice from the blocklist before the current period rolls over, so `currentPeriod() + 1` still selects the period marked claimed during rescue. The attached E2E reproduction performs those supported operations in that order and verifies both failed release paths.

First, [`rescueWithdrawFromBlocklisted`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L875) transfers all withdrawal shares for the selected period to Bob, zeroes Alice's shares, and marks Alice's bucket claimed:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

[`removeFromBlocklist`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L546-L563) restores Alice's ordinary access by changing only `isBlocklisted[Alice]`. It neither clears nor rotates Alice's withdrawal-bucket state. The repository's original blocklist test also verifies that an account can resume ordinary vault use after removal from the blocklist.

Alice's next call to [`withdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L746-L778) passes the blocklist checks. The internal [`_requestWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1026) then selects `currentPeriod() + 1`, appends a fresh entitlement, raises the pending liability, and burns Alice's shares without checking `isWithdrawClaimed[period][receiver]`:

```solidity
uint256 period = currentPeriod() + 1;
// ...
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
```

Because the period has not changed since rescue, this append targets the exact `(P, Alice)` key whose terminal flag is already `true`.

Once `P` matures, [`claimWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L780-L811) checks that flag before calculating Alice's amount or transferring assets:

```solidity
address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

The rescue function performs the same terminal-flag check at lines 862–866 before moving withdrawal shares. Re-blocklisting Alice therefore cannot recover the second entitlement through another rescue.

The attached `immunefi-poc.md` supplies a complete local-EVM reproduction, including the exact checkout, dependency revision, complete Forge configuration, complete Solidity test, execution command, expected result, and assertion-by-assertion interpretation. The verified command completes with one passing test and zero failures.

The minimal remediation is to reject a withdrawal request before any accounting mutation or share burn when `isWithdrawClaimed[currentPeriod() + 1][receiver]` is already true. A generation-specific entitlement identifier would also solve the lifetime mismatch while permitting deliberate reuse of the same receiver-period pair.

#### Impact Details

The E2E test freezes exactly **10,000 fXRP represented with six decimals (`10_000e6`)** after Alice's second deposit and withdrawal. After Bob claims the rescued first withdrawal, the final state requires all of the following:

* the vault still holds `10_000e6` fXRP;
* `pendingWithdrawAssets()` still equals `10_000e6`;
* `totalAssets()` equals zero because the contract classifies the retained tokens entirely as Alice's pending withdrawal liability;
* Alice holds zero fXRP from the second withdrawal;
* Alice's mature `claimWithdraw(P)` reverts with `AlreadyClaimedPeriod(P)`; and
* a role-authorized rescue to a fresh recovery beneficiary also reverts with `AlreadyClaimedPeriod(P)`.

These conditions prove permanent freezing of Alice's 10,000 fXRP under the current contract paths. Maturity cannot clear the terminal marker, removing Alice from the blocklist changes only her blocklist status, and adding Alice back to the blocklist leaves rescue blocked by the same marker. The contract contains no function that rotates this `(period, receiver)` key or reopens its claim state for the newly appended entitlement.

The frozen amount scales with the otherwise-valid second withdrawal and remains bounded by Alice's available vault shares and the vault's ordinary deposit and withdrawal constraints. The demonstrated value is 10,000 fXRP. Whenever the supported compliance lifecycle completes before rollover, the second withdrawal deterministically targets the rescued bucket; once accepted there, the entitlement remains permanently inaccessible.

### References

* Assessed source revision: [`42f9ea5e43d88b35197b901acc2a8c24b314fa62`](https://github.com/immunefi-team/audit-comp-firelight/tree/42f9ea5e43d88b35197b901acc2a8c24b314fa62)
* [`FirelightVault.removeFromBlocklist`, lines 546–563](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L546-L563)
* [`FirelightVault.withdraw`, lines 746–778](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L746-L778)
* [`FirelightVault.claimWithdraw`, lines 780–811](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L780-L811)
* [`FirelightVault.rescueWithdrawFromBlocklisted`, lines 839–876](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L839-L876)
* [`FirelightVault._requestWithdraw`, lines 994–1026](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1026)
* [Original blocklist tests showing restored use after removal, lines 137–155](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/test/blocklist.js#L137-L155)
* [Original pending-withdrawal rescue tests, lines 138–181](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/test/rescue.js#L138-L181)
* [Firelight technical documentation](https://docs.firelight.finance/technical-documents)
* [Firelight common workflows](https://docs.firelight.finance/technical-documents/common-workflows)

### Proof of Concept

## End-to-end reproduction: reopening a rescued receiver permanently freezes a new withdrawal

### What this demonstrates

`FirelightVault` stores withdrawal entitlements by `(period, receiver)` and stores a single terminal `isWithdrawClaimed` flag for the same key. Blocklisting Alice, rescuing her pending withdrawal to Bob, and later removing Alice from the blocklist are supported compliance operations. When this lifecycle completes before the current period rolls over, `currentPeriod() + 1` deterministically selects the same future bucket: rescue has left Alice's flag set to `true`, while the public withdrawal path accepts a new entitlement into that bucket.

Once Alice is permitted to use the vault again, every withdrawal the vault accepts and burns shares for must remain claimable or rescuable; a terminal claimed flag must never coexist with a newly accepted entitlement. Once the period matures, `claimWithdraw` rejects the terminal flag, and re-blocklisting Alice leaves `rescueWithdrawFromBlocklisted` rejecting the same flag before moving the new entitlement. The test demonstrates that Alice's second 10,000 fXRP remains held by the vault with no working claim or rescue path.

### Prerequisites

Install the following tools before starting:

* Git
* Node.js and npm
* Foundry with `forge` available on `PATH`

The initial dependency installation requires network access. Running the completed test is local-only.

{% stepper %}
{% step %}

### Obtain the assessed source

Run:

```sh
git clone --branch v1_audit_ready --single-branch https://github.com/immunefi-team/audit-comp-firelight.git firelight-poc
cd firelight-poc
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
git rev-parse HEAD
```

The final command must print:

```
42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

{% endstep %}

{% step %}

### Install dependencies

From the repository root, run:

```sh
npm install
git clone https://github.com/foundry-rs/forge-std lib/forge-std
git -C lib/forge-std checkout 467ffd422ca01fed5797a4c766a1e4e3a5327902
```

`npm install` also applies the repository's OpenZeppelin compatibility patch through its `postinstall` script.
{% endstep %}

{% step %}

### Configure Forge

Save the following file as `foundry.toml` in the repository root:

```toml
[profile.default]
src = "contracts"
test = "test"
out = "out"
libs = ["node_modules", "lib"]
fs_permissions = [{ access = "read", path = "./artifacts" }]
skip = ["contracts/fasset/**"]
evm_version = "london"
optimizer = true
optimizer_runs = 10000
via_ir = true
auto_detect_remappings = true
remappings = [
  "forge-std/=lib/forge-std/src/",
  "@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/",
  "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/",
]
```

Confirm that the assessed contracts compile before adding the test:

```sh
forge build
```

The command must finish with `Compiler run successful!`.
{% endstep %}

{% step %}

### Add the reproduction test

Save the following file as `test/RescuedWithdrawalBucketReopen.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import "forge-std/Test.sol";

import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {FirelightVault} from "contracts/core/FirelightVault.sol";
import {MockERC20} from "contracts/test/MockERC20.sol";

contract RescuedWithdrawalBucketReopenTest is Test {
    uint48 private constant PERIOD_DURATION = 1 days;
    uint256 private constant WITHDRAWAL_AMOUNT = 10_000e6;

    address private constant ALICE = address(0xA11CE);
    address private constant BOB = address(0xB0B);
    address private constant RECOVERY_BENEFICIARY = address(0xCAFE);
    address private constant COMPLIANCE = address(0xC011);
    address private constant RESCUER = address(0x5EC0);

    function setUp() public {
        vm.label(ALICE, "Alice");
        vm.label(BOB, "Bob");
        vm.label(RECOVERY_BENEFICIARY, "Recovery beneficiary");
        vm.label(COMPLIANCE, "Compliance");
        vm.label(RESCUER, "Rescuer");
    }

    function testReopenedRescuedBucketPermanentlyFreezesNewWithdrawal() public {
        MockERC20 asset = new MockERC20("fXRP", "fXRP", 6);
        FirelightVault implementation = new FirelightVault();
        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: address(this),
            limitUpdater: address(0),
            blocklister: COMPLIANCE,
            pauser: address(0),
            periodConfigurationUpdater: address(0),
            rescuer: RESCUER,
            depositLimit: 100_000e6,
            periodConfigurationDuration: PERIOD_DURATION
        });
        bytes memory initializationData = abi.encodeCall(
            FirelightVault.initialize,
            (asset, "Firelight Vault", "FLV", abi.encode(params))
        );
        FirelightVault vault = FirelightVault(
            address(new ERC1967Proxy(address(implementation), initializationData))
        );

        asset.mint(ALICE, 2 * WITHDRAWAL_AMOUNT);
        vm.startPrank(ALICE);
        asset.approve(address(vault), 2 * WITHDRAWAL_AMOUNT);
        vault.deposit(WITHDRAWAL_AMOUNT, ALICE);
        uint256 withdrawalPeriod = vault.currentPeriod() + 1;
        vault.withdraw(WITHDRAWAL_AMOUNT, ALICE, ALICE);
        vm.stopPrank();

        // Compliance blocklists Alice and the rescuer moves her first pending
        // withdrawal to Bob. The rescue permanently marks Alice's bucket claimed.
        vm.prank(COMPLIANCE);
        vault.addToBlocklist(ALICE);

        uint256[] memory periods = new uint256[](1);
        periods[0] = withdrawalPeriod;
        vm.prank(RESCUER);
        vault.rescueWithdrawFromBlocklisted(ALICE, BOB, periods);

        // Compliance reopens Alice while currentPeriod() + 1 still selects the
        // same bucket. Alice can deposit and request a second withdrawal into it.
        vm.prank(COMPLIANCE);
        vault.removeFromBlocklist(ALICE);

        vm.startPrank(ALICE);
        vault.deposit(WITHDRAWAL_AMOUNT, ALICE);
        vault.withdraw(WITHDRAWAL_AMOUNT, ALICE, ALICE);
        vm.stopPrank();

        assertTrue(vault.isWithdrawClaimed(withdrawalPeriod, ALICE));
        assertEq(vault.withdrawalsOf(withdrawalPeriod, ALICE), WITHDRAWAL_AMOUNT);
        assertEq(vault.pendingWithdrawAssets(), 2 * WITHDRAWAL_AMOUNT);

        vm.warp(block.timestamp + 2 * PERIOD_DURATION);

        // Bob's rescued entitlement follows the normal claim path.
        vm.prank(BOB);
        assertEq(vault.claimWithdraw(withdrawalPeriod), WITHDRAWAL_AMOUNT);

        // Alice's newly accepted entitlement is blocked by the stale terminal flag.
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, withdrawalPeriod)
        );
        vault.claimWithdraw(withdrawalPeriod);

        // Re-blocklisting Alice does not restore the administrative rescue path.
        vm.prank(COMPLIANCE);
        vault.addToBlocklist(ALICE);
        vm.prank(RESCUER);
        vm.expectRevert(
            abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, withdrawalPeriod)
        );
        vault.rescueWithdrawFromBlocklisted(ALICE, RECOVERY_BENEFICIARY, periods);

        // Alice's second 10,000 fXRP remains in the vault with no working claim
        // or rescue path. The pending liability exactly excludes it from totalAssets().
        assertEq(vault.pendingWithdrawAssets(), WITHDRAWAL_AMOUNT);
        assertEq(asset.balanceOf(address(vault)), WITHDRAWAL_AMOUNT);
        assertEq(vault.totalAssets(), 0);
        assertEq(asset.balanceOf(ALICE), 0);
        assertEq(asset.balanceOf(RECOVERY_BENEFICIARY), 0);
    }
}
```

{% endstep %}

{% step %}

### Run the proof of concept

From the repository root, run:

```sh
forge test --match-path test/RescuedWithdrawalBucketReopen.t.sol --match-test testReopenedRescuedBucketPermanentlyFreezesNewWithdrawal -vvv
```

The test should pass. Gas and timing values can vary by local Foundry version, but the important output is the named test and the `1 passed; 0 failed` result:

```
Ran 1 test for test/RescuedWithdrawalBucketReopen.t.sol:RescuedWithdrawalBucketReopenTest
[PASS] testReopenedRescuedBucketPermanentlyFreezesNewWithdrawal()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

This reproduction was locally verified with Node.js 24.13.0, npm 11.6.2, Forge 1.5.1-stable, and the Solidity 0.8.28 compiler selected by Forge. The observed run compiled successfully and reported one passing test with zero failures.
{% endstep %}

{% step %}

### Interpret the assertions

The test makes each state transition explicit:

1. Alice's first withdrawal targets future period `P`.
2. Compliance blocklists Alice, and the rescuer transfers Alice's entitlement for `P` to Bob. The rescue sets `isWithdrawClaimed[P][Alice]` to `true`.
3. Compliance removes Alice from the blocklist before the current period rolls over. `currentPeriod() + 1` therefore deterministically reuses `P` for Alice's second withdrawal.
4. `isWithdrawClaimed(P, Alice) == true` together with `withdrawalsOf(P, Alice) == 10_000e6` proves that the vault accepted a fresh entitlement into a terminal bucket. `pendingWithdrawAssets() == 20_000e6` proves that both Bob's rescued entitlement and Alice's new entitlement are accounted as pending.
5. Bob successfully claims 10,000 fXRP after the period matures. This is a positive control proving that the period is mature and the ordinary claim path works for the rescued entitlement.
6. Alice's claim reverts with `AlreadyClaimedPeriod(P)`. This proves that the stale terminal flag blocks her newly accepted entitlement before any transfer can occur.
7. After compliance blocklists Alice again, the authorized rescuer's recovery attempt also reverts with `AlreadyClaimedPeriod(P)`. This proves that the administrative recovery path cannot release the new entitlement either.
8. The final assertions require the vault to retain Alice's second 10,000 fXRP, keep exactly 10,000 fXRP in `pendingWithdrawAssets`, report zero active `totalAssets`, and transfer nothing to Alice or the recovery beneficiary. Together with both failed release paths, these assertions prove that the vault accepted and burned shares for an entitlement that can be neither claimed nor rescued.

No cleanup is required because Forge discards the local EVM after the test.
{% endstep %}
{% endstepper %}
