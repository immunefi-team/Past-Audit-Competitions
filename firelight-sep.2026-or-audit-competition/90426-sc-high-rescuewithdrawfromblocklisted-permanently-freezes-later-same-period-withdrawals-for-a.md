# 90426 sc high rescuewithdrawfromblocklisted permanently freezes later same period withdrawals for a reinstated account

## #90426 \[SC-High] rescueWithdrawFromBlocklisted permanently freezes later same-period withdrawals for a reinstated account

**Submitted on Aug 24th 2026 at 23:10:18 UTC by @moin for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90426
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

### Description

## **Firelight Critical Vulnerability Report**

### **Brief / Intro**

`FirelightVault.rescueWithdrawFromBlocklisted` uses the user-facing withdrawal claim flag as a rescue bookkeeping flag. When a blocklisted user's pending withdrawal is rescued, the function sets `isWithdrawClaimed[period][from] = true` even though `from` did not claim the funds. If the account is later removed from the blocklist and creates another withdrawal assigned to the same period, `_requestWithdraw` credits new withdrawal shares behind that stale flag. Once the withdrawal is mature, `claimWithdraw(period)` checks the flag first and reverts forever with `AlreadyClaimedPeriod`.

The result is a permanent freeze of the later withdrawal. The user's shares are already burned, the withdrawal view still reports assets owed, the assets remain counted in `pendingWithdrawAssets`, and both the normal claim path and the role-authorized rescue path are blocked by the same stale marker.

### **Affected State**

`FirelightVaultStorage` keeps the withdrawal entitlement and claim marker as separate mappings:

```solidity
// contracts/core/FirelightVaultStorage.sol:67-71
/// @notice Total shares allocated for withdrawals in a given period and a given account.
mapping(uint256 period => mapping(address account => uint256 shares)) public withdrawSharesOf;

/// @notice Indicates whether an account has claimed their withdrawal for a given period.
mapping(uint256 period => mapping(address account => bool value)) public isWithdrawClaimed;
```

The storage comment is important: `isWithdrawClaimed[p][account]` is documented as meaning the account has claimed. The rescue path writes that flag for an account that has not claimed.

### **Root Cause**

`claimWithdraw` checks `isWithdrawClaimed` before calculating the assets owed:

```solidity
// contracts/core/FirelightVault.sol:788-806
function claimWithdraw(uint256 period)
    external
    whenNotPaused
    notBlocklisted(_msgSender())
    nonReentrant
    returns (uint256 assets)
{
    if (period >= currentPeriod()) revert InvalidPeriod();

    address sender = _msgSender();
    if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

    assets = _convertToAssetsTotals(
        withdrawSharesOf[period][sender],
        withdrawShares[period],
        withdrawAssets[period],
        Math.Rounding.Floor
    );

    if (assets == 0) revert NoWithdrawalAmount(period);

    pendingWithdrawAssets -= assets;
    isWithdrawClaimed[period][sender] = true;
    ...
}
```

The rescue path moves the existing pending withdrawal to the recipient, zeroes the source balance, and then marks the source as claimed:

```solidity
// contracts/core/FirelightVault.sol:861-875
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true;

    rescuedShares[i] = _withdrawOf;
}
```

The flag is terminal. A grep over the scoped vault source finds writes only at:

```
contracts/core/FirelightVault.sol:806  isWithdrawClaimed[period][sender] = true;
contracts/core/FirelightVault.sol:870  isWithdrawClaimed[periods[i]][from] = true;
```

There is no `delete isWithdrawClaimed[...]` and no assignment to `false`.

The later withdrawal request path does not read the marker. It always assigns a new request to `currentPeriod() + 1`, burns the user's shares, and credits `withdrawSharesOf[period][receiver]`:

```solidity
// contracts/core/FirelightVault.sol:994-1028
function _requestWithdraw(
    uint256 assets,
    uint256 shares,
    address receiver,
    address owner
) private returns (uint256 ownerBalance) {
    if (receiver == address(0) || owner == address(0)) revert InvalidAddress();

    ownerBalance = balanceOf(owner);
    if (shares > ownerBalance) revert InsufficientShares();
    ownerBalance -= shares;

    address sender = _msgSender();

    uint256 period = currentPeriod() + 1;
    uint256 sharesWithdraw = _convertToSharesTotals(
        assets,
        withdrawShares[period],
        withdrawAssets[period],
        Math.Rounding.Floor
    );
    withdrawAssets[period] += assets;
    withdrawShares[period] += sharesWithdraw;
    withdrawSharesOf[period][receiver] += sharesWithdraw;

    pendingWithdrawAssets += assets;
    ...
}
```

This creates the invalid state:

```
isWithdrawClaimed[p][user] == true
withdrawSharesOf[p][user] > 0
```

Once this state exists, `claimWithdraw(p)` can never reach the asset calculation or transfer.

### **Precise Reachability**

The vulnerable period does not need to repeat. The reachable path occurs when the rescue and the later withdrawal both happen before the current period rolls over.

1. The current period is `P`.
2. Alice requests an initial partial withdrawal. `_requestWithdraw` books it to `p = P + 1`, leaving Alice with remaining shares.
3. Alice is blocklisted.
4. `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, bob, [p])`.

`rescueWithdrawFromBlocklisted` has no maturity or past-period-only restriction, so it can rescue and poison the future `P + 1` bucket while `P` is still current.

5. The rescue moves Alice's original pending withdrawal to Bob and sets `isWithdrawClaimed[p][alice] = true`.
6. Alice is removed from the blocklist before period `P` ends.
7. Still in period `P`, Alice redeems her remaining shares. `_requestWithdraw` again uses `currentPeriod() + 1`, so the fresh withdrawal is also booked to `p`.
8. When `p` becomes claimable, `claimWithdraw(p)` reverts with `AlreadyClaimedPeriod`.

This is why the bug is reachable without waiting for a period number to recur.

### **Why This Is In Scope Despite Privileged Roles**

This report does not claim that blocklisting or rescuing a blocklisted account is itself malicious. Those are documented compliance operations.

The in-scope defect is the side effect that exceeds the rescue authority:

* The documented rescue intent is to relocate existing value from a blocklisted address to a non-blocklisted recipient.
* The rescue does relocate the existing pending withdrawal correctly.
* The bug is that the rescue leaves a permanent marker that later disables a different withdrawal created after the account has been reinstated.
* At the moment the later funds are frozen, the harmed account is not blocklisted.
* No role has a documented permission to permanently disable future withdrawals of a reinstated account.
* This also bypasses the intended withdrawal-accounting constraint that a credited withdrawal balance should either be claimable after maturity or recoverable through rescue while blocklisted.

Firelight's official scope says privileged-role reports are out of scope only when the role acts within explicitly granted permissions, and remains in scope when the role exceeds its intended permissions or bypasses an intended protocol constraint. This finding is the latter: a normal rescue writes state that later freezes funds outside the rescue's documented blast radius.

### **Validation Checks**

* **Same-period reachability confirmed:** The PoC verifies that after rescue and delisting, the second redeem still books to the same target period `p` because `vault.currentPeriod() + 1 == p` before the second withdrawal request is credited.
* **Claim marker permanence confirmed:** `isWithdrawClaimed` is written `true` in `claimWithdraw` and `rescueWithdrawFromBlocklisted`, and the scoped vault never clears it with `false` or `delete`.
* **Frozen withdrawal balance confirmed:** After the second redeem, `withdrawSharesOf[p][alice] > 0` and `withdrawalsOf(p, alice)` reports `600e18` owed, proving the later withdrawal is real and not an empty accounting artifact.
* **User claim path blocked:** When the withdrawal becomes claimable, `claimWithdraw(p)` reverts with `AlreadyClaimedPeriod` before transferring any assets.
* **Role-based recovery path blocked:** Re-blocklisting Alice and calling `rescueWithdrawFromBlocklisted(alice, bob, [p])` also reverts with `AlreadyClaimedPeriod`, so the authorized rescue path cannot recover the frozen withdrawal.
* **Share rescue cannot recover it:** `rescueSharesFromBlocklisted` cannot help because Alice's vault shares were already burned when the later withdrawal request was created.
* **Time does not resolve the freeze:** The validation suite advances 12 additional periods and the same `AlreadyClaimedPeriod` revert still blocks the claim.
* **Control case passes without rescue:** A no-rescue control with two same-period withdrawal requests succeeds, proving normal same-period withdrawal accumulation is valid and the rescue marker is the cause.
* **Later-period request after rescue succeeds:** A rescued and delisted account can claim a later-period withdrawal normally, proving the bug is specifically caused by booking a fresh withdrawal behind the stale marker in the same period.

### **Impact**

**Selected in-scope impact:** Critical - Permanent freezing of funds.

The affected withdrawal is permanently inaccessible inside the current contract logic:

* the victim's vault shares are burned when the later withdrawal request is created;
* `withdrawSharesOf[p][victim]` remains non-zero;
* `withdrawalsOf(p, victim)` reports assets owed;
* `claimWithdraw(p)` always reverts before assets are calculated or transferred;
* the role-authorized recovery path `rescueWithdrawFromBlocklisted` also reverts;
* the corresponding amount remains permanently reserved in `pendingWithdrawAssets` because the blocked claim cannot decrement it.

The PoC demonstrates a concrete `600e18` frozen withdrawal after an initial `400e18` rescue. The maximum practical loss is the asset value represented by fresh withdrawal shares credited to the poisoned `(period, account)` pair. This can be as large as the user's remaining vault position for that period. There is no attacker profit in the base scenario; the severity comes from permanent principal unavailability, not theft.

Because the entitlement cannot be claimed, its corresponding assets remain reserved and excluded from active `totalAssets()` indefinitely:

```solidity
// contracts/core/FirelightVault.sol:427-429
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() - pendingWithdrawAssets;
}
```

### **References**

* Official Firelight Immunefi scope: <https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/>
* `contracts/core/FirelightVault.sol:788-806` - `claimWithdraw`
* `contracts/core/FirelightVault.sol:845-875` - `rescueWithdrawFromBlocklisted`
* `contracts/core/FirelightVault.sol:994-1028` - `_requestWithdraw`
* `contracts/core/FirelightVault.sol:1084-1086` - minimum period duration enforcement
* `contracts/core/FirelightVaultStorage.sol:39-40` - `SMALLEST_PERIOD_DURATION = 1 days`
* `contracts/core/FirelightVaultStorage.sol:67-71` - withdrawal balances and claim marker

### Proof of Concept

## **Proof of Concept**

This PoC is local-only Foundry code. It does not interact with mainnet or public testnet contracts and does not require a fork RPC.

***

### **Required Setup / Dependencies**

* Foundry installed (`forge` available).
* Firelight repository checked out at the audited commit.
* Repository dependencies installed.
* A Foundry test harness that deploys `FirelightVault`, grants `BLOCKLIST_ROLE` and `RESCUER_ROLE`, and exposes:
  * `asset`
  * `vault`
  * `alice`
  * `bob`
  * `blocklister`
  * `rescuer`
  * `depositAs(address,uint256)`
  * `warpToNextPeriod()`

In this audit workspace, the regression is implemented in a test contract extending the existing `FirelightBase` harness and importing `IFirelightVault`.

***

### **Steps to Run**

Run the minimal regression with:

```bash
FOUNDRY_OFFLINE=true \
FOUNDRY_TEST='test/Confirmed Finding tests' \
forge test \
  --match-path 'test/Confirmed Finding tests/ConfirmedFindingRegression.t.sol' \
  --match-test test_FL_S4_H01_rescuePoisonsLaterSamePeriodWithdrawal \
  -vv
```

***

### **Expected Result**

The test must pass and show that:

* the rescued account is **marked as claimed** for the withdrawal period;
* the account is later **delisted before the current period rolls**;
* a fresh withdrawal is **credited to the same period behind the stale marker**;
* `claimWithdraw(period)` reverts with `AlreadyClaimedPeriod`;
* the withdrawal remains **owed and reserved** in `pendingWithdrawAssets`;
* a second authorized rescue also reverts with `AlreadyClaimedPeriod`.

**Observed result from the local run:**

```
Ran 1 test for test/Confirmed Finding tests/ConfirmedFindingRegression.t.sol:ConfirmedFindingRegression
[PASS] test_FL_S4_H01_rescuePoisonsLaterSamePeriodWithdrawal()
Suite result: ok. 1 passed; 0 failed; 0 skipped.
```

***

### **PoC Code**

```solidity
function test_FL_S4_H01_rescuePoisonsLaterSamePeriodWithdrawal() public {
    uint256 aliceShares = depositAs(alice, 1_000e18);

    // Alice creates an initial partial withdrawal for the next period.
    vm.prank(alice);
    vault.redeem((aliceShares * 4) / 10, alice, alice);
    uint256 withdrawPeriod = vault.currentPeriod() + 1;

    // Normal compliance flow: Alice is blocklisted and her pending withdrawal is rescued.
    vm.prank(blocklister);
    vault.addToBlocklist(alice);

    uint256[] memory periods = new uint256[](1);
    periods[0] = withdrawPeriod;

    vm.prank(rescuer);
    vault.rescueWithdrawFromBlocklisted(alice, bob, periods);

    assertTrue(
        vault.isWithdrawClaimed(withdrawPeriod, alice),
        "source marked claimed by rescue"
    );

    // Alice is reinstated before the current period rolls over.
    vm.prank(blocklister);
    vault.removeFromBlocklist(alice);

    // The second withdrawal still targets the same period that was poisoned by rescue.
    assertEq(
        vault.currentPeriod() + 1,
        withdrawPeriod,
        "second request still targets rescued period"
    );

    uint256 remainingShares = vault.balanceOf(alice);

    vm.prank(alice);
    vault.redeem(remainingShares, alice, alice);

    assertEq(
        vault.withdrawSharesOf(withdrawPeriod, alice),
        600e18,
        "new withdrawal added behind stale claim marker"
    );

    assertTrue(
        vault.isWithdrawClaimed(withdrawPeriod, alice),
        "stale marker still blocks the period"
    );

    // Advance until the withdrawal should be claimable.
    while (vault.currentPeriod() <= withdrawPeriod) {
        warpToNextPeriod();
    }

    assertEq(
        vault.withdrawalsOf(withdrawPeriod, alice),
        600e18,
        "vault reports 600e18 owed to Alice"
    );

    uint256 aliceAssetBefore = asset.balanceOf(alice);

    // Alice cannot claim the newly requested withdrawal.
    vm.expectRevert(
        abi.encodeWithSelector(
            IFirelightVault.AlreadyClaimedPeriod.selector,
            withdrawPeriod
        )
    );
    vm.prank(alice);
    vault.claimWithdraw(withdrawPeriod);

    assertEq(
        asset.balanceOf(alice) - aliceAssetBefore,
        0,
        "Alice received no assets"
    );

    assertEq(
        vault.withdrawSharesOf(withdrawPeriod, alice),
        600e18,
        "later withdrawal remains stranded"
    );

    assertEq(
        vault.withdrawalsOf(withdrawPeriod, alice),
        600e18,
        "withdrawal remains owed but unclaimable"
    );

    assertGe(
        vault.pendingWithdrawAssets(),
        600e18,
        "later withdrawal remains reserved in pendingWithdrawAssets"
    );

    // A second authorized rescue cannot recover the stranded withdrawal either.
    vm.prank(blocklister);
    vault.addToBlocklist(alice);

    vm.expectRevert(
        abi.encodeWithSelector(
            IFirelightVault.AlreadyClaimedPeriod.selector,
            withdrawPeriod
        )
    );
    vm.prank(rescuer);
    vault.rescueWithdrawFromBlocklisted(alice, bob, periods);
}
```
