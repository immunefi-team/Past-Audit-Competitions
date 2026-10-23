# 88347 sc medium rescue writes a terminal claimed flag onto a still open withdrawal bucket permanently trapping the account s next withdrawal

**Submitted on Aug 12th 2026 at 20:07:23 UTC by @johanw for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88347
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` at `FirelightVault.sol:870` on a withdrawal bucket that is **still open for new entries**. `_requestWithdraw` writes to `currentPeriod() + 1` (`:1008`), which is the only bucket a fresh request can land in, so a user blocklisted right after requesting can only ever be rescued on an open bucket.

If that account is later removed from the blocklist while the same period is still running and queues a new withdrawal, the new withdrawal can never be claimed. `claimWithdraw` checks the flag at `:794` and reverts `AlreadyClaimedPeriod` forever. The shares were already burned inside `_requestWithdraw` at `:1025`, so the loss is realised at request time and the user only discovers it two periods later.

No AccessControl role can repair it. The rescue cannot undo its own damage: the guard at `:864` refuses a second rescue of the same `from`. Nothing in the contract clears the flag.

The fix is a one-line deletion, and this report ships the counterfactual that proves it is safe: a byte-identical copy of the vault with only line 870 removed, plus 7 adversarial tests against it.

**This is a variant of OpenZeppelin M-02, not a restatement of it.** M-02 described this exact sink, in these words: "their shares would be burned during the call to `_requestWithdraw`, but the subsequent call to `claimWithdraw` would revert due to the `isWithdrawClaimed` check." Its trigger was a `periodDuration` change causing period-number reuse, it is marked Resolved, and that trigger is genuinely dead today because `_addPeriodConfiguration:1096` keeps `startingPeriod` continuous and `:1093` forces `newEpoch >= nextPeriodEnd()`, so period numbers are monotonic. The rescue path is nowhere in M-02, and the M-02 fix does nothing for it. The delta is spelled out in full below.

There is no attacker and this report claims none. Two individually correct privileged operations compose into unrecoverable state for an unprivileged third party, with no revert and no signal.

## Vulnerability Details

**Contracts:**

* `contracts/core/FirelightVault.sol` - `rescueWithdrawFromBlocklisted()` (L845-876, the defect at L870, the guard added by an earlier fix at L865), `claimWithdraw()` (L788-811, the open-bucket bound at L791, the flag read at L794, the flag write at L806), `_requestWithdraw()` (L994-1028, the target bucket at L1008, the share burn at L1025)
* `contracts/core/FirelightVaultStorage.sol` - `isWithdrawClaimed` (L71)

Named in the competition's Assets in Scope table.

**Bug Class:** State-machine defect. A terminal, never-cleared flag is written to a mutable bucket that is still accepting entries, so a later legitimate entry into that bucket is silently unclaimable.

**Severity:** Medium. CVSS 4.0 vector: `CVSS:4.0/AV:N/AC:L/AT:P/PR:H/UI:P/VC:N/VI:H/VA:H/SC:N/SI:N/SA:N` = 5.8, computed with the `cvss` library rather than estimated. The filed severity and the CVSS band agree. `AT:P` for the operational sequence inside one period. `PR:H` because the blocklist and rescue calls require BLOCKLIST\_ROLE and RESCUER\_ROLE, and CVSS measures the privileges of the party performing the action even when that party is acting correctly. `UI:P` because the victim's own routine `withdraw` is what destroys the funds. `VA:H` for permanent unavailability of that user's funds. Selected impact is Permanent freezing of funds; that impact normally sits higher, and the severity is calibrated to Medium because the frozen amount is one user's single request rather than pooled funds, other LPs are provably unharmed, and the trigger needs a specific operational sequence.

## Root Cause

`claimWithdraw` can never set the flag on an open bucket, because it is bounded to closed ones:

```solidity
// contracts/core/FirelightVault.sol:791
if (period >= currentPeriod()) revert InvalidPeriod();
```

The rescue has no such bound, and writes the same terminal flag to any period the caller names:

```solidity
// contracts/core/FirelightVault.sol:862-871
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;      // <- terminal flag on a bucket still open
```

A fresh withdrawal can only ever land in the open bucket:

```solidity
// contracts/core/FirelightVault.sol:1008 and 1017-1025
uint256 period = currentPeriod() + 1;
...
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
...
_update(owner, address(0), shares);              // <- shares burned here, loss realised
```

so the claim is then permanently refused:

```solidity
// contracts/core/FirelightVault.sol:794
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

`isWithdrawClaimed` has exactly two writers in the contract, `:806` and `:870`, and both only ever set it to `true`. No function clears it.

**The team already recognised this hazard class and fixed half of it.** Commit `a2def47`, "Fix rescuing withdrawal to a user that already claimed them", added the `to`-side guard now at L865:

```diff
- if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
+ if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
+ if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
```

That addresses a stale flag on the destination. The source side, where the rescue itself *sets* the flag on a still-open bucket, was never addressed.

**Sequence, every step a documented operation:**

1. Period 1. Alice queues a withdrawal of 500 USDC. It lands in bucket 2, and her shares are burned.
2. Still period 1. Compliance screening flags alice, so BLOCKLIST\_ROLE blocklists her and RESCUER\_ROLE runs `rescueWithdrawFromBlocklisted` on bucket 2, which is the only bucket her request could be in. Her bucket shares move to the recovery address and `isWithdrawClaimed[2][alice]` is set.
3. Still period 1. The flag turns out to be a false positive, so BLOCKLIST\_ROLE calls `removeFromBlocklist`, a first-class function with its own event.
4. Alice, in good standing, queues another 500 USDC withdrawal. It lands in bucket 2 again, because bucket 2 is still the open bucket. Her shares are burned again. The call succeeds with no revert and no warning.
5. Period 3. The recovery address claims its 500 USDC normally. Alice's `claimWithdraw(2)` reverts `AlreadyClaimedPeriod(2)`, and will do so forever.

## Proof of Concept

See the Proof of Concept field below for the full runnable Foundry tests and output.

## Impact

An unprivileged user permanently loses the funds they committed to a withdrawal, having already surrendered the corresponding shares. In the PoC that is 500 USDC of a 1,000 USDC position, and the amount is whatever the user queues in that window.

The failure is silent in both directions that matter. The user's `withdraw` succeeds, so nothing signals that the position is already doomed, and the contract state that would reveal it, `isWithdrawClaimed(period, account)`, is a public getter no interface would think to read before withdrawing. The user discovers the loss two periods later at claim time, when it is already terminal.

No AccessControl role can repair it. Re-blocklisting and rescuing again is refused by the guard at L864, so the rescue is barred from undoing its own damage, and the position ends up both unclaimable and unrescuable.

**Stated so it is not overread:**

* Other LPs are unharmed. The share burn at `:1025` and `pendingWithdrawAssets += assets` at `:1019` are consistent, so share price does not move and `totalAssets()` cannot underflow. This is not a protocol-wide loss and is not insolvency.
* The loss is bounded to the amount queued in that one window. The user can withdraw normally in any later period.
* A secondary and *conditional* effect: `pendingWithdrawAssets` carries a reservation for a claim that can never execute, so `totalAssets()` under-reports. This is not unconditional. `payout` decrements `pendingWithdrawAssets` at `:960` and rewrites `withdrawAssets[capturePeriod + 1]` at `:954`, so an incident payout that drains that bucket inside its payout window clears the drift. The victim stays frozen either way.
* Recovery by contract upgrade is possible. The production vault uses the Transparent Upgradeable Proxy pattern, so the ProxyAdmin owner could upgrade and clear the mapping. The PoC deploys a bare `ERC1967Proxy` with no admin and does not model that escape.

## Limitations, stated up front

* The trigger needs the un-blocklisting. Every writer of `withdrawSharesOf` was checked: only `_requestWithdraw:1017`, which credits `receiver`, and the rescue itself. `withdraw:765` and `redeem:727` both carry `notBlocklisted(receiver)`, so while the account is blocklisted nothing can credit it. There is no alternative path.
* The window is the remainder of the current period. `SMALLEST_PERIOD_DURATION` is 1 day, so the window can be short.
* A rescue that targets an already-closed bucket is safe, because `currentPeriod()` is monotonic and no closed bucket can receive a new entry. That boundary is asserted by its own test.
* The PoC grants one address every role. In production BLOCKLIST\_ROLE and RESCUER\_ROLE may be held separately, which makes the sequence a two-party operation rather than a one-party one.

## Distinctness From Disclosed Issues

**OpenZeppelin M-02, the closest prior art, named first on purpose.** M-02 describes this sink in the same words used above, and is marked Resolved. Its trigger was an increase to `periodDuration` causing `currentPeriod()` to underreport and a period number to be reused. That trigger is dead in the current design: `_addPeriodConfiguration:1096` derives `startingPeriod` continuously from the previous configuration and `:1093` requires `newEpoch >= nextPeriodEnd()`, so period numbers only ever advance and a closed bucket never re-opens. The rescue path is absent from M-02, and the M-02 remediation does nothing about it. Because M-02 is Resolved rather than unfixed, the ineligibility rule for unfixed audit findings does not apply.

**Macro, 2026-07-29.** All 18 findings reviewed. None mentions the rescue path, `isWithdrawClaimed`, `claimWithdraw`, or unclaimable withdrawals.

**The November 2025 Immunefi competition** ran on the legacy vault, which ships this identical code at `contracts/legacy/FirelightVaultPredeposits.sol:760`. Of its published reports, the only two touching this function are a gas note on repeated array access inside `rescueWithdrawFromBlocklisted` and an accuracy note on the `withdrawalsOf()` view. Neither is this defect.

**Known issues.** The three listed known issues concern the first-loss buffer's peg, the first-loss buffer's custody model, and aggregate claims not being capped on-chain. None touches the blocklist or rescue path.

## Recommended Fix

The minimal fix is to delete line 870:

```solidity
// contracts/core/FirelightVault.sol:869-870
withdrawSharesOf[periods[i]][from] = 0;
- isWithdrawClaimed[periods[i]][from] = true;
```

Line 869 already zeroes the source's bucket balance, and a zero balance alone makes `claimWithdraw` revert `NoWithdrawalAmount` at `:803`. The flag write adds no protection on any of the three read sites, and it costs two things: the trap described here, and the permanent disqualification of that account from ever being re-rescued for that period. The Proof of Concept field ships a byte-identical copy of the vault with only that line removed and 7 adversarial tests against it, covering double rescue, duplicate periods in one call, rescue chains and re-rescue after a re-queue. All pass, and share conservation holds.

Two alternatives, if removing the flag write is unwelcome:

1. **Mirror the bound that already makes the claim path safe.** Add `if (periods[i] > currentPeriod()) revert InvalidPeriod();` to the rescue loop. `claimWithdraw:791` carries exactly this bound, which is precisely why the claim path can never flag an open bucket. A dedicated test asserts that property.
2. **Defence in depth.** Make `_requestWithdraw` revert when `isWithdrawClaimed[period][receiver]` is already set. That converts a silent permanent loss into a revert, and is worth adding even alongside the deletion.

## Supporting Materials

* `poc-test/RescueTrapsLaterWithdrawal.t.sol` - the defect, the redundancy observation, and the closed-bucket scope boundary. 3 tests.
* `poc-test/RefuteLine870.t.sol` - the counterfactual suite against a vault with line 870 deleted, proving the fix is safe and that the accumulator drift is clearable by `payout`. 8 tests.
* `poc-test/FirelightVaultNoFlag.sol` - generated from the in-scope source by a single `sed`, given in the Proof of Concept field, so no modified contract needs to be trusted.

## Proof of Concept

## Proof of Concept

Two suites, 11 tests, all passing. Self-contained: the vault and the mock token are deployed inside the test, so everything runs offline with no fork, no RPC endpoint and no API key, and nothing touches a deployed contract or real funds.

The second suite is the part that matters most. It runs against a copy of the in-scope vault with **only line 870 removed and nothing else changed**, which is what turns "the flag write is redundant" from an assertion into evidence. That copy is produced from the in-scope source by one `sed`, given below, so no modified contract has to be taken on trust.

### Setup

Requires Foundry and git only.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git firelight
cd firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62

mkdir -p foundry-lib poc-test
git clone --depth 1 --branch v5.2.0 https://github.com/OpenZeppelin/openzeppelin-contracts.git foundry-lib/openzeppelin-contracts
git clone --depth 1 --branch v5.4.0 https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable.git foundry-lib/openzeppelin-contracts-upgradeable
git clone --depth 1 --branch v1.9.6 https://github.com/foundry-rs/forge-std.git foundry-lib/forge-std
```

Save the `foundry.toml` below to the repository root, and the two test files below to `poc-test/`. Then generate the counterfactual vault, which differs from the in-scope source only by the deleted line and the import paths and contract name needed to compile it from `poc-test/`:

```bash
sed -e 's#from "\./#from "../contracts/core/#g' \
    -e 's/contract FirelightVault is/contract FirelightVaultNoFlag is/' \
    -e '/isWithdrawClaimed\[periods\[i\]\]\[from\] = true;/d' \
    contracts/core/FirelightVault.sol > poc-test/FirelightVaultNoFlag.sol
```

The OpenZeppelin versions are the pair `package.json` pins: `"@openzeppelin/contracts": "5.2.0"` under `overrides`, and `"@openzeppelin/contracts-upgradeable": "~5.4.0"`. `foundry.toml` skips `contracts/fasset/**` and `contracts/oracle/**`, the only sources importing the `flare-smart-contracts` npm packages.

### Run

```bash
forge test --match-path 'poc-test/RescueTraps*' -vv
forge test --match-path 'poc-test/RefuteLine870*' -vv
```

**Scenario:** a vault with a 7 day period and two 1,000 USDC depositors. Alice queues 500 USDC in period 1, is blocklisted and rescued while bucket 2 is still open, is un-blocklisted in the same period, and queues 500 USDC again. Every timestamp is derived from a fixed `t0` rather than from `block.timestamp` after a warp, because solc common-subexpression eliminates `block.timestamp` across `vm.warp` under `via_ir` and a naive capture silently reads the pre-warp value.

### Result (actual forge output)

```
Compiling 1 files with Solc 0.8.28
Solc 0.8.28 finished in 13.16s
Compiler run successful!
Ran 3 tests for poc-test/RescueTrapsLaterWithdrawal.t.sol:RescueTrapsLaterWithdrawal
[PASS] test_control_zeroedBucketSharesAlreadyBlockAClaimWithoutTheFlag() (gas: 355936)
[PASS] test_rescueOnAnOpenBucketPermanentlyTrapsALaterWithdrawal() (gas: 541129)
Logs:
  assets locked, unclaimable forever (6d): 5e8
  pendingWithdrawAssets stranded      (6d): 5e8
  totalAssets under-report           (6d): 5e8
  victim shares were burned at request time, so the loss is realised
[PASS] test_scope_rescueOnAClosedBucketIsSafe() (gas: 660800)
Logs:
  closed-bucket rescue is safe, alice claimed (6d): 5e8
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 2.85ms (1.91ms CPU time)
Ran 1 test suite in 12.56ms (2.85ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
No files changed, compilation skipped
Ran 8 tests for poc-test/RefuteLine870.t.sol:RefuteLine870
[PASS] test_attack_doubleRescueSamePeriodStillReverts() (gas: 406989)
[PASS] test_attack_duplicatePeriodInOneCallStillReverts() (gas: 808145)
[PASS] test_attack_fromCannotClaimAfterRescue() (gas: 426017)
[PASS] test_attack_payoutCanDrainTheStuckAccumulator() (gas: 510221)
Logs:
  paid out (6d): 2e9
  withdrawAssets[2] after payout (6d): 0e0
  pendingWithdrawAssets after payout (6d): 0e0
[PASS] test_attack_reRescueAfterRequeueConservesValue() (gas: 963323)
Logs:
  original: re-queued position is unrescuable AND unclaimable
[PASS] test_attack_rescueChainAndToSideGuard() (gas: 851614)
[PASS] test_claimPathCannotFlagAnOpenBucket() (gas: 347750)
[PASS] test_patchedContractHasNoTrap() (gas: 481205)
Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 4.95ms (6.84ms CPU time)
Ran 1 test suite in 12.48ms (4.95ms CPU time): 8 tests passed, 0 failed, 0 skipped (8 total tests)
```

### What each test establishes

`test_rescueOnAnOpenBucketPermanentlyTrapsALaterWithdrawal` is the defect. It asserts the trap, then asserts there is no repair through any role: re-blocklisting and calling the rescue again reverts `AlreadyClaimedPeriod`, so the rescue cannot undo its own damage and the position is left both unclaimable and unrescuable.

`test_scope_rescueOnAClosedBucketIsSafe` states the boundary of the finding rather than leaving it to be discovered. A rescue targeting an already-closed bucket is harmless, because `currentPeriod()` is monotonic and a closed bucket can never receive a new entry. Alice's later withdrawal lands in bucket 3 and claims normally.

`test_control_zeroedBucketSharesAlreadyBlockAClaimWithoutTheFlag` shows that a zero bucket balance alone already causes `claimWithdraw` to revert `NoWithdrawalAmount`, which is the observation the remediation rests on. It is suggestive rather than conclusive on its own, which is why the second suite exists.

`test_patchedContractHasNoTrap` is the counterfactual: the identical sequence against the vault with line 870 deleted, where alice claims her second withdrawal normally.

`test_attack_doubleRescueSamePeriodStillReverts`, `test_attack_duplicatePeriodInOneCallStillReverts`, `test_attack_fromCannotClaimAfterRescue` and `test_attack_rescueChainAndToSideGuard` are the adversarial checks on the proposed fix. On the patched contract a second rescue of the same source still reverts, now via `NoWithdrawalAmount` from the zeroed balance instead of `AlreadyClaimedPeriod`; a duplicate period inside one call still reverts because the balance is re-read each iteration; the rescued source still cannot claim; and the `to`-side guard added by commit `a2def47` still fires on a rescue chain. Nothing is lost by the deletion.

`test_attack_reRescueAfterRequeueConservesValue` shows the patched contract allowing the re-queued position to be rescued, with share conservation holding across all three accounts and `pendingWithdrawAssets` draining to zero. The same test logs that on the original contract that position is both unrescuable and unclaimable.

`test_claimPathCannotFlagAnOpenBucket` proves the alternative remediation is the right shape: the `period >= currentPeriod()` bound at `claimWithdraw:791` is load-bearing, and is exactly what the rescue lacks.

`test_attack_payoutCanDrainTheStuckAccumulator` is included because it argues against part of my own case. It shows an incident payout decrementing `pendingWithdrawAssets` back to zero, which is why the accumulator drift is reported above as conditional rather than permanent. The victim remains frozen throughout it.

A first run compiles the whole tree, so the compile line differs from a cached run. Gas figures may vary slightly by Foundry version; the asserted values do not.

### Build configuration, foundry.toml

```toml
[profile.default]
src = "contracts"
out = "foundry-out"
libs = ["foundry-lib"]
test = "poc-test"
cache_path = "foundry-cache"
solc = "0.8.28"
optimizer = true
optimizer_runs = 10000
via_ir = true
evm_version = "cancun"
skip = ["contracts/fasset/**", "contracts/oracle/**"]
remappings = [
  "@openzeppelin/contracts/=foundry-lib/openzeppelin-contracts/contracts/",
  "@openzeppelin/contracts-upgradeable/=foundry-lib/openzeppelin-contracts-upgradeable/contracts/",
  "forge-std/=foundry-lib/forge-std/src/",
]
```

### poc-test/RescueTrapsLaterWithdrawal.t.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";
import {MockERC20} from "../contracts/test/MockERC20.sol";

/// @notice `rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` on a
///         withdrawal bucket that is STILL OPEN for new entries. `_requestWithdraw` writes to
///         `currentPeriod() + 1`, and the rescue can target exactly that bucket. If the rescued
///         account is later removed from the blocklist while the same period is still running and
///         queues a new withdrawal, that new withdrawal can never be claimed: `claimWithdraw`
///         reverts `AlreadyClaimedPeriod` forever.
///
///         `isWithdrawClaimed` has only two writers in the whole contract, `claimWithdraw:806` and
///         `rescueWithdrawFromBlocklisted:870`, and both only ever set it to true. No function
///         clears it, so no AccessControl role can repair the state; only a proxy upgrade could.
///
///         Every role behaves correctly throughout. The blocklist and rescue are used exactly as
///         designed, the un-blocklisting is a normal correction, and the victim is unprivileged.
contract RescueTrapsLaterWithdrawal is Test {
    address admin = makeAddr("admin");
    address alice = makeAddr("alice");      // unprivileged victim
    address rescueDest = makeAddr("rescueDest");
    address bystander = makeAddr("bystander");

    MockERC20 asset;
    FirelightVault vault;

    uint48 constant PERIOD = 7 days;
    uint256 constant DEPOSIT = 1_000e6;
    uint256 constant FIRST_WITHDRAW = 500e6;
    uint256 constant SECOND_WITHDRAW = 500e6;

    uint48 t0;

    function setUp() public {
        vm.warp(1_800_000_000);
        t0 = uint48(block.timestamp);

        asset = new MockERC20("USD Stable", "USD", 6);

        FirelightVault impl = new FirelightVault();
        IFirelightVault.InitParams memory vp = IFirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: 1_000_000_000e6,
            periodConfigurationDuration: PERIOD
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(impl),
                    abi.encodeCall(
                        FirelightVault.initialize, (IERC20(address(asset)), "Firelight", "FLT", abi.encode(vp))
                    )
                )
            )
        );

        _deposit(alice, DEPOSIT);
        _deposit(bystander, DEPOSIT);
    }

    function _deposit(address who, uint256 amount) internal {
        asset.mint(who, amount);
        vm.startPrank(who);
        asset.approve(address(vault), amount);
        vault.deposit(amount, who);
        vm.stopPrank();
    }

    /// @notice The defect. All timestamps derive from `t0` because solc common-subexpression
    ///         eliminates `block.timestamp` across `vm.warp` under `via_ir`.
    function test_rescueOnAnOpenBucketPermanentlyTrapsALaterWithdrawal() public {
        // ---- period 1: alice queues a withdrawal, which lands in bucket 2 ----
        vm.warp(t0 + PERIOD + 1);
        assertEq(vault.currentPeriod(), 1, "period 1");

        vm.prank(alice);
        vault.withdraw(FIRST_WITHDRAW, alice, alice);

        uint256 bucket = 2;
        assertGt(vault.withdrawSharesOf(bucket, alice), 0, "alice holds bucket 2 shares");
        assertEq(vault.pendingWithdrawAssets(), FIRST_WITHDRAW, "one withdrawal queued");

        // ---- still period 1: alice is blocklisted, and the rescue runs as designed ----
        vm.startPrank(admin);
        vault.addToBlocklist(alice);
        uint256[] memory periods = new uint256[](1);
        periods[0] = bucket;
        vault.rescueWithdrawFromBlocklisted(alice, rescueDest, periods);
        vm.stopPrank();

        assertEq(vault.withdrawSharesOf(bucket, alice), 0, "alice's bucket shares moved out");
        assertTrue(vault.isWithdrawClaimed(bucket, alice), "the flag the rescue set");

        // Bucket 2 is still being filled: it only closes when period 1 ends.
        assertEq(vault.currentPeriod(), 1, "bucket 2 is still open for new entries");

        // ---- still period 1: the blocklisting is reversed, a normal correction ----
        vm.prank(admin);
        vault.removeFromBlocklist(alice);

        // ---- alice queues again, in good standing, into that same still-open bucket ----
        uint256 pendingBefore = vault.pendingWithdrawAssets();
        vm.prank(alice);
        vault.withdraw(SECOND_WITHDRAW, alice, alice);

        assertGt(vault.withdrawSharesOf(bucket, alice), 0, "alice holds bucket 2 shares again");
        assertEq(
            vault.pendingWithdrawAssets(), pendingBefore + SECOND_WITHDRAW, "the new withdrawal is accounted"
        );

        // ---- period 3: bucket 2 is claimable ----
        vm.warp(t0 + 3 * PERIOD + 1);
        assertEq(vault.currentPeriod(), 3, "period 3");

        // The rescue destination is paid normally, so the vault is solvent and the bucket works.
        vm.prank(rescueDest);
        uint256 rescuedPaid = vault.claimWithdraw(bucket);
        assertEq(rescuedPaid, FIRST_WITHDRAW, "rescued claim pays out");

        // Alice cannot claim the withdrawal she made while in good standing.
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, bucket));
        vault.claimWithdraw(bucket);

        // ---- no AccessControl role can repair it ----
        // Re-blocklisting and rescuing again is refused by the guard at line 864, so the rescue
        // cannot undo its own damage. Nothing in the contract clears the flag. A proxy upgrade
        // could, since the production vault is behind a Transparent proxy; this test deploys a
        // bare ERC1967Proxy with no admin, so it does not model that escape.
        vm.startPrank(admin);
        vault.addToBlocklist(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, bucket));
        vault.rescueWithdrawFromBlocklisted(alice, bystander, periods);
        vault.removeFromBlocklist(alice);
        vm.stopPrank();

        // The shares are still recorded against alice, and the assets are still reserved.
        uint256 lockedShares = vault.withdrawSharesOf(bucket, alice);
        uint256 lockedAssets = vault.withdrawalsOf(bucket, alice);
        assertGt(lockedShares, 0, "shares remain assigned to alice");
        assertEq(lockedAssets, SECOND_WITHDRAW, "assets remain reserved for a claim that cannot happen");

        // ---- secondary, conditional: the reservation is stranded ----
        // `pendingWithdrawAssets` is decremented only by `claimWithdraw` and by `payout`. The claim
        // can never run, so the reservation persists unless an incident payout happens to drain
        // this bucket inside its payout window. It is not unconditionally permanent.
        assertEq(vault.pendingWithdrawAssets(), SECOND_WITHDRAW, "reservation stranded, absent a payout");
        assertEq(
            vault.totalAssets(),
            asset.balanceOf(address(vault)) - SECOND_WITHDRAW,
            "totalAssets under-reports by the stranded amount"
        );

        console2.log("assets locked, unclaimable forever (6d): %e", lockedAssets);
        console2.log("pendingWithdrawAssets stranded      (6d): %e", vault.pendingWithdrawAssets());
        console2.log("totalAssets under-report           (6d): %e", SECOND_WITHDRAW);
        console2.log("victim shares were burned at request time, so the loss is realised");
    }

    /// @notice The fix is a one-line deletion, and this proves nothing is lost by it.
    ///         Line 869 already zeroes `withdrawSharesOf[period][from]`, and a zero bucket balance
    ///         alone makes `claimWithdraw` revert `NoWithdrawalAmount`. The flag write on line 870
    ///         adds no protection; it only makes the state terminal.
    function test_control_zeroedBucketSharesAlreadyBlockAClaimWithoutTheFlag() public {
        vm.warp(t0 + PERIOD + 1);

        vm.prank(alice);
        vault.withdraw(FIRST_WITHDRAW, alice, alice);

        uint256 bucket = 2;
        vm.warp(t0 + 3 * PERIOD + 1);

        // bystander never queued anything in this bucket: zero shares, flag never set.
        assertEq(vault.withdrawSharesOf(bucket, bystander), 0, "no shares");
        assertFalse(vault.isWithdrawClaimed(bucket, bystander), "flag not set");

        vm.prank(bystander);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.NoWithdrawalAmount.selector, bucket));
        vault.claimWithdraw(bucket);
    }

    /// @notice Scope. A rescue that targets an ALREADY CLOSED bucket is safe, because no new entry
    ///         can be written to it. The defect is specific to the still-open `currentPeriod() + 1`
    ///         bucket, and this states that boundary rather than leaving it to be found.
    function test_scope_rescueOnAClosedBucketIsSafe() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        vault.withdraw(FIRST_WITHDRAW, alice, alice);
        uint256 bucket = 2;

        // Move into period 2. Bucket 2 is now closed: new withdrawals go to bucket 3.
        vm.warp(t0 + 2 * PERIOD + 1);
        assertEq(vault.currentPeriod(), 2, "period 2");

        vm.startPrank(admin);
        vault.addToBlocklist(alice);
        uint256[] memory periods = new uint256[](1);
        periods[0] = bucket;
        vault.rescueWithdrawFromBlocklisted(alice, rescueDest, periods);
        vault.removeFromBlocklist(alice);
        vm.stopPrank();

        // Alice queues again; it lands in bucket 3, which carries no stale flag.
        vm.prank(alice);
        vault.withdraw(SECOND_WITHDRAW, alice, alice);
        assertEq(vault.withdrawSharesOf(bucket, alice), 0, "nothing added to the closed bucket");
        assertGt(vault.withdrawSharesOf(3, alice), 0, "new entry landed in bucket 3");

        vm.warp(t0 + 4 * PERIOD + 1);
        vm.prank(alice);
        uint256 paid = vault.claimWithdraw(3);
        assertEq(paid, SECOND_WITHDRAW, "bucket 3 claims normally");
        console2.log("closed-bucket rescue is safe, alice claimed (6d): %e", paid);
    }
}
```

### poc-test/RefuteLine870.t.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {FirelightVaultNoFlag} from "./FirelightVaultNoFlag.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";
import {MockERC20} from "../contracts/test/MockERC20.sol";

/// @notice ADVERSARIAL control suite. The claimed remediation is "delete line 870".
///         These tests try to BREAK the patched contract: double rescue, duplicate periods in one
///         call, rescue chains, from-claims-after-rescue, and share/asset conservation.
contract RefuteLine870 is Test {
    address admin = makeAddr("admin");
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address carol = makeAddr("carol");
    address dave = makeAddr("dave");

    MockERC20 asset;
    FirelightVaultNoFlag v;      // patched: line 870 removed
    FirelightVault orig;         // unpatched, for differential comparison

    uint48 constant PERIOD = 7 days;
    uint256 constant DEP = 1_000e6;
    uint256 constant W = 500e6;

    uint48 t0;

    function setUp() public {
        vm.warp(1_800_000_000);
        t0 = uint48(block.timestamp);
        asset = new MockERC20("USD Stable", "USD", 6);

        IFirelightVault.InitParams memory vp = IFirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: 1_000_000_000e6,
            periodConfigurationDuration: PERIOD
        });
        bytes memory init = abi.encodeCall(
            FirelightVault.initialize, (IERC20(address(asset)), "Firelight", "FLT", abi.encode(vp))
        );
        v = FirelightVaultNoFlag(address(new ERC1967Proxy(address(new FirelightVaultNoFlag()), init)));
        orig = FirelightVault(address(new ERC1967Proxy(address(new FirelightVault()), init)));

        _dep(address(v), alice, DEP);
        _dep(address(v), bob, DEP);
        _dep(address(orig), alice, DEP);
        _dep(address(orig), bob, DEP);
    }

    function _dep(address vault, address who, uint256 amt) internal {
        asset.mint(who, amt);
        vm.startPrank(who);
        asset.approve(vault, amt);
        FirelightVault(vault).deposit(amt, who);
        vm.stopPrank();
    }

    function _p(uint256 x) internal pure returns (uint256[] memory a) {
        a = new uint256[](1);
        a[0] = x;
    }

    // ---------------------------------------------------------------------
    // ATTACK 1a: with 870 gone, can the rescuer rescue the SAME period twice
    //            and mint value out of thin air?
    // ---------------------------------------------------------------------
    function test_attack_doubleRescueSamePeriodStillReverts() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        v.withdraw(W, alice, alice);

        vm.startPrank(admin);
        v.addToBlocklist(alice);
        v.rescueWithdrawFromBlocklisted(alice, bob, _p(2));
        // second rescue of the same still-blocklisted account, same period
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.NoWithdrawalAmount.selector, uint256(2)));
        v.rescueWithdrawFromBlocklisted(alice, carol, _p(2));
        vm.stopPrank();

        assertEq(v.withdrawSharesOf(2, bob), W, "bob holds exactly the rescued amount");
        assertEq(v.withdrawSharesOf(2, carol), 0, "carol got nothing");
        assertEq(v.withdrawSharesOf(2, alice), 0, "alice zeroed");
    }

    // ---------------------------------------------------------------------
    // ATTACK 1b: duplicate period inside ONE rescue call. Line 870 was the
    //            thing that stopped iteration 2 in the original. Does the
    //            patched version double-credit?
    // ---------------------------------------------------------------------
    function test_attack_duplicatePeriodInOneCallStillReverts() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        v.withdraw(W, alice, alice);

        uint256[] memory dup = new uint256[](2);
        dup[0] = 2;
        dup[1] = 2;

        vm.startPrank(admin);
        v.addToBlocklist(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.NoWithdrawalAmount.selector, uint256(2)));
        v.rescueWithdrawFromBlocklisted(alice, bob, dup);
        vm.stopPrank();

        // and the original reverts too, just with a different selector
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        orig.withdraw(W, alice, alice);
        vm.startPrank(admin);
        orig.addToBlocklist(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, uint256(2)));
        orig.rescueWithdrawFromBlocklisted(alice, bob, dup);
        vm.stopPrank();
    }

    // ---------------------------------------------------------------------
    // ATTACK 1c: after being rescued, can `from` claim the SAME period and
    //            get paid twice out of the bucket?
    // ---------------------------------------------------------------------
    function test_attack_fromCannotClaimAfterRescue() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        v.withdraw(W, alice, alice);

        vm.startPrank(admin);
        v.addToBlocklist(alice);
        v.rescueWithdrawFromBlocklisted(alice, bob, _p(2));
        v.removeFromBlocklist(alice);
        vm.stopPrank();

        vm.warp(t0 + 3 * PERIOD + 1);

        // the zeroed bucket balance alone blocks the claim - this is the whole
        // remediation argument, tested on the PATCHED contract this time.
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.NoWithdrawalAmount.selector, uint256(2)));
        v.claimWithdraw(2);

        vm.prank(bob);
        assertEq(v.claimWithdraw(2), W, "bob gets it, exactly once");

        vm.prank(bob);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, uint256(2)));
        v.claimWithdraw(2);
    }

    // ---------------------------------------------------------------------
    // ATTACK 1d: rescue chain A -> B -> C, plus the to-side guard from
    //            commit a2def47 must still fire.
    // ---------------------------------------------------------------------
    function test_attack_rescueChainAndToSideGuard() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        v.withdraw(W, alice, alice);

        vm.startPrank(admin);
        v.addToBlocklist(alice);
        v.rescueWithdrawFromBlocklisted(alice, bob, _p(2)); // A -> B
        v.addToBlocklist(bob);
        v.rescueWithdrawFromBlocklisted(bob, carol, _p(2)); // B -> C
        vm.stopPrank();

        assertEq(v.withdrawSharesOf(2, carol), W, "carol holds it");
        assertEq(v.withdrawSharesOf(2, bob), 0, "bob zeroed");

        vm.warp(t0 + 3 * PERIOD + 1);
        vm.prank(carol);
        assertEq(v.claimWithdraw(2), W, "carol claims once");

        // to-side guard: carol has now claimed period 2, so she can never be a
        // rescue destination for period 2 again. That guard is untouched.
        vm.startPrank(admin);
        v.removeFromBlocklist(alice);
        vm.stopPrank();
        vm.prank(dave);
        // give dave a bucket-4 position, blocklist him, try to rescue INTO carol
        vm.stopPrank();
        _dep(address(v), dave, DEP);
        vm.prank(dave);
        v.withdraw(W, dave, dave);
        vm.startPrank(admin);
        v.addToBlocklist(dave);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, uint256(2)));
        v.rescueWithdrawFromBlocklisted(dave, carol, _p(2));
        vm.stopPrank();
    }

    // ---------------------------------------------------------------------
    // ATTACK 1e: the exact sequence the original contract BLOCKS. With 870
    //            gone the rescuer can rescue a re-queued position. Prove that
    //            this does NOT create value: conservation of shares + assets.
    // ---------------------------------------------------------------------
    function test_attack_reRescueAfterRequeueConservesValue() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        v.withdraw(W, alice, alice);

        vm.startPrank(admin);
        v.addToBlocklist(alice);
        v.rescueWithdrawFromBlocklisted(alice, bob, _p(2));
        v.removeFromBlocklist(alice);
        vm.stopPrank();

        // alice re-queues into the SAME still-open bucket 2
        vm.prank(alice);
        v.withdraw(W, alice, alice);

        // re-blocklist and rescue again -> allowed on the patched contract
        vm.startPrank(admin);
        v.addToBlocklist(alice);
        v.rescueWithdrawFromBlocklisted(alice, carol, _p(2));
        vm.stopPrank();

        // conservation: per-account shares sum to the bucket total
        assertEq(
            v.withdrawSharesOf(2, bob) + v.withdrawSharesOf(2, carol) + v.withdrawSharesOf(2, alice),
            v.withdrawShares(2),
            "no shares created or destroyed"
        );

        uint256 vaultBefore = asset.balanceOf(address(v));
        vm.warp(t0 + 3 * PERIOD + 1);
        vm.prank(bob);
        uint256 b = v.claimWithdraw(2);
        vm.prank(carol);
        uint256 c = v.claimWithdraw(2);

        assertEq(b + c, 2 * W, "exactly the two requests, no more");
        assertEq(asset.balanceOf(address(v)), vaultBefore - 2 * W, "vault paid exactly that");
        assertEq(v.pendingWithdrawAssets(), 0, "accumulator fully drained, no residue");

        // the original contract cannot do this at all
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        orig.withdraw(W, alice, alice);
        vm.startPrank(admin);
        orig.addToBlocklist(alice);
        orig.rescueWithdrawFromBlocklisted(alice, bob, _p(2));
        orig.removeFromBlocklist(alice);
        vm.stopPrank();
        vm.prank(alice);
        orig.withdraw(W, alice, alice);
        vm.startPrank(admin);
        orig.addToBlocklist(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, uint256(2)));
        orig.rescueWithdrawFromBlocklisted(alice, carol, _p(2));
        vm.stopPrank();
        console2.log("original: re-queued position is unrescuable AND unclaimable");
    }

    // ---------------------------------------------------------------------
    // The trap itself, on the patched contract: gone.
    // ---------------------------------------------------------------------
    function test_patchedContractHasNoTrap() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        v.withdraw(W, alice, alice);

        vm.startPrank(admin);
        v.addToBlocklist(alice);
        v.rescueWithdrawFromBlocklisted(alice, bob, _p(2));
        v.removeFromBlocklist(alice);
        vm.stopPrank();

        vm.prank(alice);
        v.withdraw(W, alice, alice);

        vm.warp(t0 + 3 * PERIOD + 1);
        vm.prank(alice);
        assertEq(v.claimWithdraw(2), W, "alice claims normally on the patched contract");
        vm.prank(bob);
        assertEq(v.claimWithdraw(2), W, "and so does the rescue destination");
        assertEq(v.pendingWithdrawAssets(), 0, "no residue");
    }

    // ---------------------------------------------------------------------
    // ATTACK 6: the report claims pendingWithdrawAssets is inflated FOREVER.
    //           payout() also decrements it (FirelightVault.sol:960). Does a
    //           payout quietly repair the drift, or the victim?
    // ---------------------------------------------------------------------
    function test_attack_payoutCanDrainTheStuckAccumulator() public {
        vm.startPrank(admin);
        orig.grantRole(orig.PAYOUT_ROLE(), admin);
        orig.grantRole(orig.PAYOUT_ALLOWLIST_ROLE(), admin);
        orig.addToPayoutAllowlist(carol);
        vm.stopPrank();

        // build the trap in bucket 2 during period 1
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        orig.withdraw(W, alice, alice);
        vm.startPrank(admin);
        orig.addToBlocklist(alice);
        orig.rescueWithdrawFromBlocklisted(alice, bob, _p(2));
        orig.removeFromBlocklist(alice);
        vm.stopPrank();
        vm.prank(alice);
        orig.withdraw(W, alice, alice);

        assertEq(orig.pendingWithdrawAssets(), 2 * W, "both requests reserved");
        assertEq(orig.withdrawAssets(2), 2 * W, "bucket 2 holds both");

        // incident captured in period 1 -> payout draws from withdrawAssets[2]
        vm.prank(admin);
        uint256 paid = orig.payout(carol, 2_000e6, uint48(t0 + PERIOD + 1));
        console2.log("paid out (6d): %e", paid);
        console2.log("withdrawAssets[2] after payout (6d): %e", orig.withdrawAssets(2));
        console2.log("pendingWithdrawAssets after payout (6d): %e", orig.pendingWithdrawAssets());

        // the accumulator drift IS cleared - so "permanently inflated" is conditional
        assertEq(orig.pendingWithdrawAssets(), 0, "accumulator drained by payout");

        // but the victim is NOT repaired: still flagged, still cannot claim
        vm.warp(t0 + 3 * PERIOD + 1);
        assertTrue(orig.isWithdrawClaimed(2, alice), "flag survives the payout");
        assertGt(orig.withdrawSharesOf(2, alice), 0, "shares still assigned to alice");
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, uint256(2)));
        orig.claimWithdraw(2);
    }

    // ---------------------------------------------------------------------
    // Scope check the report leans on: claimWithdraw can NEVER set the flag on
    // an open bucket, because of the period bound at FirelightVault.sol:791.
    // ---------------------------------------------------------------------
    function test_claimPathCannotFlagAnOpenBucket() public {
        vm.warp(t0 + PERIOD + 1);
        vm.prank(alice);
        orig.withdraw(W, alice, alice);
        // bucket 2 is open (currentPeriod == 1). Claiming it is refused outright.
        vm.prank(alice);
        vm.expectRevert(FirelightVault.InvalidPeriod.selector);
        orig.claimWithdraw(2);
        vm.prank(alice);
        vm.expectRevert(FirelightVault.InvalidPeriod.selector);
        orig.claimWithdraw(1); // currentPeriod itself: also refused
    }
}
```
