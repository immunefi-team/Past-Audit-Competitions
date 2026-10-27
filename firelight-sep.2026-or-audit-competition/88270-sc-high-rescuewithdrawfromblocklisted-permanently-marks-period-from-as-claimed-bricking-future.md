# 88270 sc high rescuewithdrawfromblocklisted permanently marks period from as claimed bricking future withdrawals for that period and stranding the funds in pendingwithdrawassets

**Submitted on Aug 12th 2026 at 14:37:32 UTC by @Web3Vinay for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88270
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

A rescue of a *pending* (not-yet-claimable) withdrawal will cause **permanent, unrecoverable loss of the withdrawn assets for the rescued user, and a permanent understatement of `totalAssets()` for every other LP**, because `rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` on a period slot that can still be re-filled by a later `withdraw`/`redeem`, and `claimWithdraw` treats that flag as final.

The `isWithdrawClaimed` flag is used as a *slot-consumed* marker, but the underlying slot (`withdrawSharesOf[period][account]`) is **not** single-use: `_requestWithdraw` credits `withdrawSharesOf[currentPeriod() + 1][receiver] += ...` on every request. Any withdrawal request the user books into the same period number after the rescue lands on a slot that is already flagged "claimed" and can never be paid out — not by the user, and not by `RESCUER_ROLE` either.

Therefore, a legitimate use of rescueWithdrawFromBlocklisted function can poison the user's period slot and permanently brick a later legitimate withdrawal into that same period

## Vulnerability Details

[FirelightVault.sol:845-876](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845-L876) — `rescueWithdrawFromBlocklisted` marks `from` as claimed without any payout ever occurring, and without any restriction on the period being already claimable:

```solidity
// FirelightVault.sol:861-873
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
@>    isWithdrawClaimed[periods[i]][from] = true;   // <-- :870  "claimed" without a payout
    ...
}
```

There is no check that `periods[i] < currentPeriod()`. In practice the rescue target *is* the pending period, because that is where an outstanding withdrawal lives: `_requestWithdraw` always books into `currentPeriod() + 1` (`contracts/core/FirelightVault.sol:1008`) and `claimWithdraw` only unlocks a period once `period < currentPeriod()` (`:791`). So the rescued slot is, by construction, still open for new deposits of claim rights.

`_requestWithdraw` re-fills the very slot that was just flagged:

```solidity
// FirelightVault.sol:1008-1017
uint256 period = currentPeriod() + 1;
...
@> withdrawSharesOf[period][receiver] += sharesWithdraw;   // :1017  slot is additive, not single-use
@> pendingWithdrawAssets += assets;                        // :1019
```

`claimWithdraw` then rejects that brand-new, fully paid-for claim right forever, because it consults the account-level flag before it ever looks at the balance:

```solidity
// FirelightVault.sol:793-803
address sender = _msgSender();
@> if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // :794
assets = _convertToAssetsTotals(withdrawSharesOf[period][sender], ...);
```

And the only escape hatch is closed by the rescue function's own guard at `:864` — a second rescue of the same `(period, from)` reverts `AlreadyClaimedPeriod`, so `RESCUER_ROLE` cannot move the stuck entry out either. `withdrawSharesOf`/`isWithdrawClaimed` have no other writer (`_requestWithdraw` at `:1017` and `rescueWithdrawFromBlocklisted` at `:868-870` are the only two), so the state is terminal.

Second-order damage: the stranded assets were added to `pendingWithdrawAssets` at `:1019` and are only ever released by `claimWithdraw`'s `pendingWithdrawAssets -= assets` at `:805` (or `payout()` at `:960`, which is out of window by then). Since the claim can never execute, the counter is permanently inflated, and `totalAssets()` is permanently deflated for *every* remaining LP:

```solidity
// FirelightVault.sol:427-429
return super.totalAssets() - pendingWithdrawAssets;
```

## Internal Pre-conditions

1. A user has an open withdrawal request for the pending period `P = currentPeriod() + 1` (`withdrawSharesOf[P][user] > 0`, `isWithdrawClaimed[P][user] == false`).
2. `BLOCKLIST_ROLE` blocklists that user (compliance flag, false positive, key-compromise report, etc.).
3. `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(user, dest, [P])` — the documented, intended use of the function for an outstanding withdrawal.
4. `BLOCKLIST_ROLE` later un-blocklists the user (flag cleared / false positive resolved) **while the chain is still inside period `P - 1`**, i.e. while `currentPeriod() + 1 == P`.
5. The user files any new `withdraw`/`redeem` while still in period `P - 1`.

All four privileged steps are squarely inside documented authority. The trigger in step 5 is a **permissionless, ordinary user action** — the user does not know their period slot has been poisoned, and there is no view function or event that would tell them (`maxClaim`-style helper at `:465-474` reads `withdrawSharesOf` only and does not consult `isWithdrawClaimed`).

## External Pre-conditions

None.

## Attack Path

Concrete numbers from the PoC (6-decimal asset, 7-day periods):

1. Bob (unrelated LP) deposits `100,000` fXRP.
2. Alice deposits `100,000` fXRP and redeems all of it → `withdrawSharesOf[P][alice] = 100,000e6`, `pendingWithdrawAssets = 100,000e6`.
3. `BLOCKLIST_ROLE` blocklists Alice.
4. `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, aliceNew, [P])` → `withdrawSharesOf[P][aliceNew] = 100,000e6`, `withdrawSharesOf[P][alice] = 0`, **`isWithdrawClaimed[P][alice] = true`** (`:870`) — no assets moved.
5. `BLOCKLIST_ROLE` un-blocklists Alice (false positive resolved). Still inside period `P - 1`.
6. Alice deposits `50,000` fXRP and redeems → `withdrawSharesOf[P][alice] = 50,000e6`, `pendingWithdrawAssets = 150,000e6`. **The flag from step 4 is still `true`.**
7. Two period rolls; `P` becomes claimable. `aliceNew` claims `100,000` fXRP successfully.
8. Alice calls `claimWithdraw(P)` → reverts `AlreadyClaimedPeriod(P)` (`:794`). She receives nothing.
9. Recovery attempt: re-blocklist Alice and call `rescueWithdrawFromBlocklisted(alice, aliceNew, [P])` → reverts `AlreadyClaimedPeriod(P)` at `:864`. **No recovery path exists.**

## Impact

* **Direct, permanent loss of `50,000` fXRP for Alice** in the PoC — the amount is unbounded and equals whatever the user withdraws into the poisoned period.
* **Irreversibly bricked state**: neither the user nor `RESCUER_ROLE` nor any admin function can clear `isWithdrawClaimed[P][alice]` or move `withdrawSharesOf[P][alice]`. There is no setter, and the vault is upgradeable only via a full implementation upgrade.
* **Accounting broken for every other LP**: `pendingWithdrawAssets` is permanently inflated by the stranded amount, so `totalAssets()` (`:427-429`) is permanently understated by the same amount. In the PoC the vault holds `150,000e6` raw but reports `totalAssets() = 100,000e6`. This silently:
  * depresses the share price for all remaining holders (`_convertToAssetsTotals` denominators),
  * consumes `depositLimit` headroom that will never be freed (`maxDeposit` at `:371-378`),
  * shrinks `payout()`'s `currentActiveAssets` cap (`:912`) forever, reducing the cover the vault can honour.
* Broken invariant, exactly: `claimWithdraw`'s one-shot flag is asserted to be per-*claim*, but it is in fact a permanent per-*slot* marker on a slot that is re-fillable — a booked, fully-funded claim right can exist with `isWithdrawClaimed == true` and zero payout ever made.

**Note on the pre-declared exclusions:** the privileged actors here all stay inside documented authority. The result — an ordinary user's later funds irreversibly bricked and every other LP's share accounting permanently corrupted — is the "privileged role's action produces a result outside its documented authority (breaks accounting for others, bricks state irreversibly)" carve-out, not centralization risk.

## Reference

`contracts/core/FirelightVault.sol:845-876` — [rescueWithdrawFromBlocklisted](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845-L876) `contracts/core/FirelightVault.sol:788-811` — [claimWithdraw](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L811)

## Proof of Concept

## Proof of Concept

The project uses Hardhat, but I used Foundry as an independent testing harness for the PoC.

### Run

```
forge test --match-test test_poc_rescueBricksFutureClaimForSamePeriod -vvvv
```

### Test Code

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

// Standalone PoC — needs only forge-std, OpenZeppelin (already a repo dependency)
// and the in-scope contract `contracts/core/FirelightVault.sol`.
//
//   forge test --match-path "audit-data/poc-tests/rescue-bricks-claim.t.sol" -vvv
//
// (or drop this file into the configured `test` directory, e.g. `audit-data/foundry/`)

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {FirelightVault} from "../../contracts/core/FirelightVault.sol";

contract PocAsset {
    string public name = "Mock fXRP";
    string public symbol = "fXRP";
    uint8 public constant decimals = 6;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
        emit Transfer(address(0), to, amount);
    }

    function approve(address s, uint256 a) external returns (bool) {
        allowance[msg.sender][s] = a;
        emit Approval(msg.sender, s, a);
        return true;
    }

    function transfer(address to, uint256 a) external returns (bool) {
        _t(msg.sender, to, a);
        return true;
    }

    function transferFrom(address f, address to, uint256 a) external returns (bool) {
        uint256 al = allowance[f][msg.sender];
        if (al != type(uint256).max) {
            require(al >= a, "allowance");
            allowance[f][msg.sender] = al - a;
        }
        _t(f, to, a);
        return true;
    }

    function _t(address f, address to, uint256 a) internal {
        require(balanceOf[f] >= a, "balance");
        balanceOf[f] -= a;
        balanceOf[to] += a;
        emit Transfer(f, to, a);
    }
}

contract RescueBricksClaimPoC is Test {
    FirelightVault internal vault;
    PocAsset internal asset;

    address internal admin = address(0xA11CE);
    address internal blocklister = address(0xB10CC);
    address internal rescuer = address(0x2E5C4E);
    address internal pauser = address(0xBA05E);
    address internal limitUpdater = address(0x11311);
    address internal periodUpdater = address(0xBE710D);

    address internal alice = address(0xA71CE);      // honest user, blocklisted by mistake
    address internal aliceNew = address(0xA71CE2);  // rescue destination
    address internal bob = address(0xB0B);          // unrelated LP, suffers the accounting damage

    uint48 internal constant PERIOD_DURATION = 7 days;
    uint256 internal constant DEPOSIT_LIMIT = 50_000_000e6;

    function setUp() public {
        vm.warp(1_700_000_000);
        asset = new PocAsset();

        FirelightVault impl = new FirelightVault();
        FirelightVault.InitParams memory p = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: limitUpdater,
            blocklister: blocklister,
            pauser: pauser,
            periodConfigurationUpdater: periodUpdater,
            rescuer: rescuer,
            depositLimit: DEPOSIT_LIMIT,
            periodConfigurationDuration: PERIOD_DURATION
        });
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(impl),
            abi.encodeCall(
                FirelightVault.initialize,
                (IERC20(address(asset)), "Staked fXRP", "stfXRP", abi.encode(p))
            )
        );
        vault = FirelightVault(address(proxy));

        address[3] memory us = [alice, aliceNew, bob];
        for (uint256 i = 0; i < us.length; i++) {
            asset.mint(us[i], 1_000_000e6);
            vm.prank(us[i]);
            asset.approve(address(vault), type(uint256).max);
        }
    }

    function test_poc_rescueBricksFutureClaimForSamePeriod() public {
        // ---- unrelated LP provides liquidity ----
        vm.prank(bob);
        vault.deposit(100_000e6, bob);

        // ---- 1. Alice deposits and files a withdraw request ----
        vm.startPrank(alice);
        vault.deposit(100_000e6, alice);
        uint256 P = vault.currentPeriod() + 1; // the pending withdrawal period
        vault.redeem(vault.balanceOf(alice), alice, alice);
        vm.stopPrank();

        assertEq(vault.withdrawSharesOf(P, alice), 100_000e6, "alice request booked");
        assertFalse(vault.isWithdrawClaimed(P, alice));

        // ---- 2. Alice is blocklisted (compliance flag) and her pending
        //         withdrawal is rescued to a fresh address ----
        vm.prank(blocklister);
        vault.addToBlocklist(alice);

        uint256[] memory periods = new uint256[](1);
        periods[0] = P;
        vm.prank(rescuer);
        vault.rescueWithdrawFromBlocklisted(alice, aliceNew, periods);

        // FirelightVault.sol:870 -- `from` is flagged "claimed" even though nothing was paid
        assertTrue(vault.isWithdrawClaimed(P, alice), "alice flagged claimed by the rescue");
        assertEq(vault.withdrawSharesOf(P, alice), 0);
        assertEq(vault.withdrawSharesOf(P, aliceNew), 100_000e6);

        // ---- 3. The flag turns out to be a false positive; Alice is un-blocklisted
        //         and resumes normal use of the vault, still inside period P-1 ----
        vm.prank(blocklister);
        vault.removeFromBlocklist(alice);
        assertEq(vault.currentPeriod() + 1, P, "still the same period, new requests land in P");

        vm.startPrank(alice);
        vault.deposit(50_000e6, alice);
        uint256 stranded = vault.redeem(vault.balanceOf(alice), alice, alice);
        vm.stopPrank();

        // brand-new, fully paid-for claim right lands on an already-flagged slot
        assertGt(vault.withdrawSharesOf(P, alice), 0, "new entry booked for alice");
        assertTrue(vault.isWithdrawClaimed(P, alice), "...but the slot is still flagged claimed");
        emit log_named_uint("assets Alice can never claim", stranded);

        // ---- 4. Period P becomes claimable ----
        vm.warp(uint256(vault.currentPeriodEnd()) + 1);
        vm.warp(uint256(vault.currentPeriodEnd()) + 1);
        assertGt(vault.currentPeriod(), P);

        // the rescue destination is paid normally
        vm.prank(aliceNew);
        uint256 paidToNew = vault.claimWithdraw(P);
        assertGt(paidToNew, 0);

        // ---- 5. Alice can NEVER claim her second, legitimate withdrawal ----
        uint256 aliceBalBefore = asset.balanceOf(alice);
        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(bytes4(keccak256("AlreadyClaimedPeriod(uint256)")), P)
        );
        vault.claimWithdraw(P);
        assertEq(asset.balanceOf(alice), aliceBalBefore, "alice received nothing");

        // ---- 6. There is no recovery path, not even for RESCUER_ROLE:
        //         rescueWithdrawFromBlocklisted reverts at FirelightVault.sol:864 ----
        vm.prank(blocklister);
        vault.addToBlocklist(alice);
        vm.prank(rescuer);
        vm.expectRevert(
            abi.encodeWithSelector(bytes4(keccak256("AlreadyClaimedPeriod(uint256)")), P)
        );
        vault.rescueWithdrawFromBlocklisted(alice, aliceNew, periods);

        // ---- 7. Damage is not only Alice's: the stranded amount stays inside
        //         pendingWithdrawAssets forever, so totalAssets() (and therefore every
        //         remaining LP's share price) is permanently understated by it. ----
        assertGe(vault.pendingWithdrawAssets(), stranded, "stranded assets stuck in pendingWithdrawAssets");

        uint256 rawBalance = asset.balanceOf(address(vault));
        uint256 reported = vault.totalAssets();
        assertEq(rawBalance - reported, vault.pendingWithdrawAssets());

        emit log_named_uint("vault raw balance          ", rawBalance);
        emit log_named_uint("vault totalAssets()        ", reported);
        emit log_named_uint("pendingWithdrawAssets stuck", vault.pendingWithdrawAssets());

        // Bob is the only remaining LP; his shares are backed by assets the vault
        // refuses to count, and the stuck amount can never be released.
        assertGt(vault.balanceOf(bob), 0);
        assertEq(vault.pendingWithdrawAssets(), stranded, "exactly Alice's second request is stuck");
    }
}
```

### Result

```
Ran 1 test for audit-data/foundry/Inv03_PoC_RescueBricksClaim.t.sol:RescueBricksClaimPoC
[PASS] test_poc_rescueBricksFutureClaimForSamePeriod() (gas: 886681)
Logs:
  assets Alice can never claim: 50000000000
  vault raw balance          : 150000000000
  vault totalAssets()        : 100000000000
  pendingWithdrawAssets stuck: 50000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Every assertion in the test asserts the *broken* behaviour, so a green run is the proof: Alice's `50,000e6` is unclaimable by her, unrescuable by `RESCUER_ROLE`, and permanently subtracted from `totalAssets()` (`150,000e6` held vs `100,000e6` reported).
