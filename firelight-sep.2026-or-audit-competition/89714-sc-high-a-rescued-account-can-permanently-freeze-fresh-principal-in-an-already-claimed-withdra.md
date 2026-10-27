# 89714 sc high a rescued account can permanently freeze fresh principal in an already claimed withdrawal bucket

**Submitted on Aug 20th 2026 at 17:53:56 UTC by @go4ko for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89714
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal to a recovery address and permanently sets `isWithdrawClaimed[period][from] = true`. This terminal marker is appropriate for the entitlement that was rescued. However, it applies to the entire `(period, account)` storage slot, including future withdrawal requests that do not yet exist.

If the account is subsequently removed from the blocklist while the same withdrawal-request period is still open, it can deposit fresh principal and initiate a new withdrawal to itself. `_requestWithdraw` burns the new shares and credits the already-claimed account-period slot without checking `isWithdrawClaimed`. When the period becomes claimable, `claimWithdraw` always reverts with `AlreadyClaimedPeriod`. Re-blocklisting the account cannot recover the entitlement because the rescue function rejects the same marker.

The end-to-end PoC isolates the defect entirely within the in-scope V2 implementation and removes any partial-rescue assumption: it completely rescues all old shares and pending withdrawals, unblocks the empty account, and then demonstrates that an entirely fresh 3,000 FXRP deposit is burned into an unrecoverable withdrawal. All rescued old capital is successfully claimed by the recovery address, leaving exactly the fresh 3,000 FXRP permanently held by the vault with zero share supply.

The vulnerability does not depend on the V1-to-V2 upgrade, inherited V1 state, an empty deployment, or an initialization artifact. Every state required for the failure is created through ordinary V2 operations after V2 is active. The PoC therefore deploys the real V2 vault directly, as the upgrade mechanism is irrelevant to the vulnerable state transition.

This sequence does not rely on a blocklister or rescuer making a mistake. Adding an account to the blocklist, completely rescuing its assets, and later removing it from the blocklist are legitimate role operations exposed by the production contract. The repository's own blocklist tests expressly call `removeFromBlocklist` and then verify that the unblocked account can resume ordinary token operations. The vulnerability is the missing terminal-marker check when that restored account later requests a withdrawal.

## Vulnerability Details

A pending-withdrawal rescue moves the existing withdrawal shares to the recovery address, zeros the original entitlement, and terminally marks the original account-period pair as claimed:

```solidity
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

The marker is not scoped to the rescued entitlement or to a withdrawal-bucket generation. It is a permanent Boolean associated with that period and address.

Removing the address from the blocklist is an explicitly supported operation:

```solidity
function removeFromBlocklist(address account)
    external
    onlyRole(BLOCKLIST_ROLE)
    onlyBlocklisted(account)
{
    isBlocklisted[account] = false;
    emit RemovedFromBlocklist(account);
}
```

After removal, the account can use ordinary vault operations again. A fresh withdrawal is assigned to `currentPeriod() + 1`, and `_requestWithdraw` credits the receiver without checking whether that receiver's target slot is already terminal:

```solidity
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
_update(owner, address(0), shares);
```

Consequently, the fresh shares are burned and the underlying is removed from active assets even though the resulting entitlement can never pass the claim guard:

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

The rescue path cannot serve as recovery. If the account is blocklisted again, `rescueWithdrawFromBlocklisted` reaches its own `isWithdrawClaimed[period][from]` check and reverts before moving the newly recorded entitlement. No function clears the marker, creates a new generation for the account-period slot, or returns the burned shares.

### Attack / Failure Sequence

1. A user deposits 4,000 FXRP into the V2 vault and requests withdrawal of 1,000 FXRP into period `P`.
2. The account is legitimately blocklisted.
3. `RESCUER_ROLE` performs a complete compliance rescue:
   * all 3,000 remaining active shares are moved to a recovery address; and
   * the 1,000-FXRP pending withdrawal for period `P` is moved to the recovery address.
4. The rescue leaves the original account with no shares and no pending withdrawal, while setting `isWithdrawClaimed[P][user] = true`.
5. `BLOCKLIST_ROLE` legitimately removes the account from the blocklist before the current request period ends.
6. The user deposits an entirely fresh 3,000 FXRP and receives 3,000 new shares.
7. The user calls the ordinary `withdraw(3_000 FXRP, user, user)`. Because the target is still period `P`, `_requestWithdraw` adds the entitlement to `withdrawSharesOf[P][user]` despite the existing terminal marker and burns all 3,000 shares.
8. After the withdrawal delay, the recovery address claims all 4,000 FXRP of rescued old capital.
9. The user's claim for the fresh 3,000 FXRP reverts with `AlreadyClaimedPeriod(P)`.
10. Re-blocklisting and attempting another rescue also reverts with `AlreadyClaimedPeriod(P)`.
11. The vault is left holding exactly 3,000 FXRP as `pendingWithdrawAssets`, with zero active assets and zero share supply.

A third party with an ordinary vault-share allowance can also trigger the defective withdrawal on behalf of the affected owner and use it as griefing. No protocol role is needed for the state transition that burns the fresh shares into the terminal bucket.

### Why the privileged-role exclusion does not remove the issue

The blocklister and rescuer do not act maliciously or exceed their authority. They perform the documented compliance lifecycle and completely move all pre-existing assets away from the blocklisted account. The PoC expressly proves that no old shares or pending withdrawal remain before unblocking.

The vulnerable action happens afterward: the in-scope V2 vault accepts an unprivileged withdrawal into a storage slot that its own rescue logic has made permanently unclaimable. This bypasses the intended claimed-period constraint rather than deriving harm from arbitrary role misuse.

### Known-Issue Distinction

The supplied Macro audit describes `BLOCKLIST_ROLE` and `RESCUER_ROLE` as a standard compliance rescue pattern and notes their intentional power to move a blocklisted user's assets. It does not identify reuse of a rescued account's terminal withdrawal slot after unblocking.

This issue does not rely on a malicious rescuer redirecting funds. All old assets arrive at the intended recovery address and are successfully claimed. Only new principal deposited after the account has been unblocked becomes frozen.

The issue also does not appear in the program's supplied known-issues list.

## Impact Details

The direct state consequence is permanent freezing of funds: shares are burned, the assets remain included in `pendingWithdrawAssets`, and both available recovery paths permanently revert. The frozen amount is not limited to dust; once the affected state exists, the account can burn its entire fresh balance into the poisoned bucket.

The exact in-scope impact is **Critical - Permanent freezing of funds**. The required operational sequence is uncommon: the same account must be blocklisted, have a future pending withdrawal rescued, be unblocked, and make another withdrawal before the current request period rolls over. This lowers likelihood, but it does not make the resulting loss temporary or recoverable. Once the fresh withdrawal is accepted, the user's shares are burned and every ordinary claim and rescue path permanently rejects the entitlement. For likelihood calibration, the report is submitted at a **High threat level** rather than claiming Critical severity.

Low likelihood does not make the behavior dependent on privileged error. Both `addToBlocklist` and `removeFromBlocklist` are explicit production functions restricted to `BLOCKLIST_ROLE`; the rescue functions are explicit production functions restricted to `RESCUER_ROLE`; and the project's tests verify that removing an address from the blocklist restores its ability to transact. Every privileged call in the PoC succeeds exactly as designed and all pre-existing assets are recovered. Only the subsequent unprivileged withdrawal enters an impossible-to-claim state.

## Recommended Remediation

At minimum, reject a withdrawal request when its target receiver-period pair is already terminal:

```diff
uint256 period = currentPeriod() + 1;
+ if (isWithdrawClaimed[period][receiver]) {
+     revert AlreadyClaimedPeriod(period);
+ }

uint256 sharesWithdraw = _convertToSharesTotals(
    assets,
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);
```

A more robust design is to replace the permanent account-period Boolean with withdrawal-bucket generations or nonces. Rescue should terminalize only the entitlement that was moved, not every future entitlement that happens to reuse the same account and period.

Add regression coverage for:

* complete share and withdrawal rescue followed by unblock;
* fresh deposit and withdrawal before the period rolls;
* delegated withdrawal by an approved share spender; and
* re-blocklist/rescue recovery attempts.

## References

* [`FirelightVault.removeFromBlocklist`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L556-L563)
* [`FirelightVault.claimWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L780-L811)
* [`FirelightVault.rescueWithdrawFromBlocklisted`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L839-L876)
* [`FirelightVault._requestWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028)
* [Blocklist tests: removal restores normal account operation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/test/blocklist.js#L137-L155)
* [Firelight: Deployments and Withdrawals](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals)
* [Supplied Macro audit](https://library-pr-440.onrender.com/library/audits/firelight-1)

## Proof of Concept

## Proof of Concept

Create `test/rescue-claimed-marker-poc.js` with the following contents.

This is a fully end-to-end V2 PoC. It deploys the real in-scope V2 `FirelightVault` and exercises the complete deposit, withdrawal request, blocklist, full rescue, unblock, fresh deposit, fresh withdrawal, claim, and failed-recovery flows. The fixture mocks only the external FXRP-compatible asset dependency.

The PoC intentionally creates all relevant state after V2 is active. The V1-to-V2 upgrade does not create, preserve, or otherwise contribute to the bug; the collision is between V2's permanent rescue marker and a later V2 withdrawal request for the same account and period.

Run:

```bash
npx hardhat test test/rescue-claimed-marker-poc.js
```

Code:

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { deployVault } = require("./setup/fixtures.js");

describe("E2E V2 PoC: rescued withdrawal marker permanently freezes fresh principal", function () {
  it("completely rescues the old position, then traps a fresh 3,000 FXRP withdrawal after unblock", async function () {
    const ctx = await loadFixture(deployVault);
    const user = ctx.users[0];
    const recovery = ctx.users[1];
    const vault = ctx.firelight_vault;
    const asset = ctx.token_contract;

    expect(await vault.contractVersion()).to.equal(2n);

    const oldDeposit = ethers.parseUnits("4000", 6);
    const oldWithdrawal = ethers.parseUnits("1000", 6);
    const oldActiveShares = oldDeposit - oldWithdrawal;
    const freshDeposit = ethers.parseUnits("3000", 6);

    // Every state transition below occurs on the real in-scope V2 FirelightVault.
    await ctx.utils.mintAndApprove(oldDeposit + freshDeposit, user);
    await vault.connect(user).deposit(oldDeposit, user.address);

    const poisonedPeriod = (await vault.currentPeriod()) + 1n;
    await vault.connect(user).withdraw(oldWithdrawal, user.address, user.address);
    expect(await vault.balanceOf(user.address)).to.equal(oldActiveShares);

    // Perform a complete, legitimate compliance rescue: both the user's active
    // shares and its pending withdrawal are moved to the recovery address.
    await vault.connect(ctx.blocklister).addToBlocklist(user.address);
    await expect(vault.connect(ctx.rescuer).rescueSharesFromBlocklisted(user.address, recovery.address))
      .to.emit(vault, "SharesRescuedFromBlocklisted")
      .withArgs(user.address, recovery.address, oldActiveShares);
    await expect(
      vault.connect(ctx.rescuer).rescueWithdrawFromBlocklisted(user.address, recovery.address, [poisonedPeriod]),
    ).to.emit(vault, "WithdrawRescuedFromBlocklisted");

    expect(await vault.balanceOf(user.address)).to.equal(0n);
    expect(await vault.withdrawalsOf(poisonedPeriod, user.address)).to.equal(0n);
    expect(await vault.isWithdrawClaimed(poisonedPeriod, user.address)).to.equal(true);

    // Unblocking is a supported operation. The account then re-enters with entirely
    // fresh principal; none of its pre-blocklist shares or withdrawals remain.
    await vault.connect(ctx.blocklister).removeFromBlocklist(user.address);
    await vault.connect(user).deposit(freshDeposit, user.address);
    expect(await vault.balanceOf(user.address)).to.equal(freshDeposit);

    // _requestWithdraw accepts the new request for the same still-open period even
    // though that (period, receiver) slot is already terminally marked as claimed.
    await expect(vault.connect(user).withdraw(freshDeposit, user.address, user.address))
      .to.emit(vault, "WithdrawRequest")
      .withArgs(user.address, user.address, user.address, poisonedPeriod, freshDeposit, freshDeposit);
    expect(await vault.balanceOf(user.address)).to.equal(0n);
    expect(await vault.withdrawalsOf(poisonedPeriod, user.address)).to.equal(freshDeposit);

    // The recovery address exits all 4,000 FXRP of legitimately rescued old capital,
    // ensuring that only the user's fresh 3,000 FXRP remains afterward.
    await vault.connect(recovery).redeem(oldActiveShares, recovery.address, recovery.address);
    await time.increase(ctx.config.period_configuration_duration * 2);
    await expect(vault.connect(recovery).claimWithdraw(poisonedPeriod))
      .to.emit(vault, "CompleteWithdraw")
      .withArgs(recovery.address, oldDeposit, poisonedPeriod);
    expect(await asset.balanceOf(recovery.address)).to.equal(oldDeposit);

    // The fresh withdrawal cannot be claimed because rescue permanently marked the
    // account-period pair. Re-blocklisting and using rescue again cannot recover it.
    await expect(vault.connect(user).claimWithdraw(poisonedPeriod))
      .to.be.revertedWithCustomError(vault, "AlreadyClaimedPeriod")
      .withArgs(poisonedPeriod);

    await vault.connect(ctx.blocklister).addToBlocklist(user.address);
    await expect(
      vault.connect(ctx.rescuer).rescueWithdrawFromBlocklisted(user.address, recovery.address, [poisonedPeriod]),
    )
      .to.be.revertedWithCustomError(vault, "AlreadyClaimedPeriod")
      .withArgs(poisonedPeriod);

    // All active and legitimately rescued capital has left. The vault holds exactly
    // the user's fresh 3,000 FXRP as permanently pending, with no shares outstanding.
    expect(await vault.totalSupply()).to.equal(0n);
    expect(await vault.totalAssets()).to.equal(0n);
    expect(await vault.pendingWithdrawAssets()).to.equal(freshDeposit);
    expect(await asset.balanceOf(await vault.getAddress())).to.equal(freshDeposit);
    expect(await asset.balanceOf(user.address)).to.equal(0n);
  });
});
```

Expected result:

```
  E2E V2 PoC: rescued withdrawal marker permanently freezes fresh principal
    ✔ completely rescues the old position, then traps a fresh 3,000 FXRP withdrawal after unblock

  1 passing
```
