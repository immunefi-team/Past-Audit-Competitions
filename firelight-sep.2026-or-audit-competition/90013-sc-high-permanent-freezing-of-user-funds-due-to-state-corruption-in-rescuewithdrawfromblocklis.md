# 90013 sc high permanent freezing of user funds due to state corruption in rescuewithdrawfromblocklisted when resolving compliance false positives

**Submitted on Aug 22nd 2026 at 13:57:17 UTC by @hunraj for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90013
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Executive Summary

The `FirelightVault` contract provides compliance tools to blocklist users and rescue their pending withdrawals. However, the `rescueWithdrawFromBlocklisted` function introduces a destructive state corruption by permanently setting the `isWithdrawClaimed` flag to `true` for the rescued period.

If an innocent user is erroneously blocklisted (a compliance false positive), rescued, and subsequently unblocklisted within the same vault period, any new withdrawal requests they make during that period will be permanently locked. When the period ends, the user cannot claim their funds, and the admin's `RESCUER_ROLE` is completely bypassed because the rescue function itself reverts on the corrupted flag. This leads to the permanent freezing of user funds.

## Vulnerability Description

In [`FirelightVault.sol`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol?utm_source=immunefi#L870), delayed withdrawals are assigned to the `currentPeriod() + 1`. When a user's account is flagged for compliance reasons, the `RESCUER_ROLE` can extract their pending withdrawals using `rescueWithdrawFromBlocklisted()`.

The vulnerability exists in how the user's state is mutated during the rescue:

```solidity
function rescueWithdrawFromBlocklisted(address from, address to, uint256[] calldata periods) external ... {
   for (uint256 i = 0; i < len; i++) {
       uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];
       if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
       // ...
       withdrawSharesOf[periods[i]][to] += _withdrawOf;
       withdrawSharesOf[periods[i]][from] = 0;
      
       // @audit ROOT CAUSE: This permanently flags the period as claimed for the user
       isWithdrawClaimed[periods[i]][from] = true;
       rescuedShares[i] = _withdrawOf;
   }
   // ...
}
```

Setting `isWithdrawClaimed` to `true` is mathematically redundant (as zeroing `withdrawSharesOf` ensures any future `claimWithdraw` calculates `0` assets and reverts with `NoWithdrawalAmount`). However, setting this flag introduces a fatal side effect if the user is unblocklisted within the same vault period.

If the compliance flag was a false positive, the `BLOCKLIST_ROLE` calls `removeFromBlocklist()`, restoring the user's access. The user resumes standard protocol activity and initiates a new `withdraw()` or `redeem()`.

Because the protocol is still in the same period, the new withdrawal is assigned to the exact same target period bucket. The protocol happily burns the user's shares and locks their assets in `pendingWithdrawAssets`, incrementing `withdrawSharesOf[period][from]`.

However, when the period advances and the user attempts to retrieve their assets via [`claimWithdraw()`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol?utm_source=immunefi#L794), the transaction reverts:

```solidity
function claimWithdraw(uint256 period) external ... {
   // ...
   if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period); // ALWAYS REVERTS
   // ...
}
```

The user's funds are now permanently frozen in the vault.

## Impact Assessment

* **Severity:** **High** (Permanent freezing of funds).
* **Likelihood:** Medium/High. Firelight's documentation notes that periods will be 30 days long. A 30-day window is more than enough time for a user to be flagged by automated compliance tools (e.g., Chainalysis), investigated, cleared as a false positive, and resume trading.
* **Bypass of Emergency Recovery:** Crucially, if the admin realizes the user's funds are stuck, they cannot use the `RESCUER_ROLE` to save them. The `rescueWithdrawFromBlocklisted` function contains the exact same `if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);` check, meaning the protocol's emergency escape hatch is also completely neutralized by the corrupted state.

## Step-by-Step Reproduction

1. **Period 10:** Alice requests a withdrawal of 5,000 USDC. It is scheduled for Period 11.
2. **Period 10:** Alice's wallet is flagged by automated off-chain compliance monitoring. Admin calls `addToBlocklist(Alice)`.
3. **Period 10:** Admin calls `rescueWithdrawFromBlocklisted(Alice, ComplianceWallet, [11])`. Alice's `isWithdrawClaimed[11]` is set to `true`.
4. **Period 10:** Compliance investigation concludes Alice's flag was a false positive. Admin calls `removeFromBlocklist(Alice)`.
5. **Period 10:** Alice resumes normal activity and requests a new withdrawal of 2,000 USDC. It is scheduled for Period 11. Her shares are burned and 2,000 USDC is locked in `pendingWithdrawAssets`.
6. **Period 11 starts:** Alice calls `claimWithdraw(11)`. The transaction reverts with `AlreadyClaimedPeriod` because of the state corruption in Step 3. Her 2,000 USDC is permanently lost.
7. **Failed Rescue:** Admin tries to blocklist Alice and rescue the 2,000 USDC. `rescueWithdrawFromBlocklisted` also reverts with `AlreadyClaimedPeriod`. The funds are dead.

## Proof of Concept

## Proof of Concept

Place the following code in `test/PoC_RescueLock.js` and execute via `npx hardhat test test/PoC_RescueLock.js`.

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers');
const { expect } = require('chai');
const { ethers } = require('hardhat');
const { deployVault } = require('./setup/fixtures.js');


describe('FirelightVault Rescue Lock PoC', function () {
   const DECIMALS = 6;
   const INITIAL_DEPOSIT = ethers.parseUnits('10000', DECIMALS);
   const FIRST_WITHDRAW = ethers.parseUnits('2000', DECIMALS);
   const SECOND_WITHDRAW = ethers.parseUnits('3000', DECIMALS);


   it('Permanently locks user funds if rescued then unblocklisted within the same period', async function () {
       const { firelight_vault, blocklister, rescuer, users, utils, config } = await loadFixture(
           deployVault.bind(null, {
               initial_deposit_limit: ethers.parseUnits('100000', DECIMALS),
               period_configuration_duration: 604800 // 1 week
           })
       );
      
       const alice = users[0];
       const complianceWallet = users[1];


       // 1. Initial Deposit
       await utils.mintAndApprove(INITIAL_DEPOSIT, alice);
       await firelight_vault.connect(alice).deposit(INITIAL_DEPOSIT, alice.address);


       // 2. Alice requests a withdrawal. Targets `currentPeriod + 1`.
       const startPeriod = await firelight_vault.currentPeriod();
       const targetPeriod = startPeriod + 1n;
       await firelight_vault.connect(alice).withdraw(FIRST_WITHDRAW, alice.address, alice.address);


       // 3. The False Positive: Alice is erroneously flagged and blocklisted
       await firelight_vault.connect(blocklister).addToBlocklist(alice.address);


       // 4. Compliance team rescues the pending withdrawal
       await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, complianceWallet.address, [targetPeriod]);


       // State Corruption: The rescue erroneously set the flag to true for Alice
       expect(await firelight_vault.isWithdrawClaimed(targetPeriod, alice.address)).to.equal(true);


       // 5. The Resolution: Compliance team clears Alice and unblocklists her
       await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address);


       // 6. Still in the SAME period, Alice resumes normal activity and requests another withdrawal
       await firelight_vault.connect(alice).withdraw(SECOND_WITHDRAW, alice.address, alice.address);


       const stuckShares = await firelight_vault.withdrawSharesOf(targetPeriod, alice.address);
       expect(stuckShares).to.be.gt(0n);


       // 7. Time advances so the target period becomes claimable
       await time.increase(config.period_configuration_duration * 2);


       // 8. The Permanent Lock: Alice tries to claim her new withdrawal, but it REVERTS
       await expect(
           firelight_vault.connect(alice).claimWithdraw(targetPeriod)
       ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(targetPeriod);


       // 9. The Failed Admin Recovery:
       // Admin realizes the mistake and tries to rescue Alice's stuck funds
       await firelight_vault.connect(blocklister).addToBlocklist(alice.address);
      
       // The rescue function also REVERTS, rendering the funds permanently dead
       await expect(
           firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, complianceWallet.address, [targetPeriod])
       ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(targetPeriod);


       console.log(`\n\t--- POC RESULTS ---`);
       console.log(`\tAlice's Stuck Withdrawal Shares : ${ethers.formatUnits(stuckShares, DECIMALS)}`);
       console.log(`\tCan Alice Claim?                : FALSE (Reverted)`);
       console.log(`\tCan Admin Rescue?               : FALSE (Reverted)`);
       console.log(`\tFunds Status                    : PERMANENTLY LOCKED\n`);
   });
});
```

## Security Recommendations

Remove the line that sets `isWithdrawClaimed[periods[i]][from] = true;` inside `rescueWithdrawFromBlocklisted()`.

This explicit flag mutation is completely unnecessary. `claimWithdraw()` determines the `assets` to send by proportionally querying the user's share balance:

```solidity
assets = _convertToAssetsTotals(withdrawSharesOf[period][sender], withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor);
if (assets == 0) revert NoWithdrawalAmount(period);
```

Because the rescue function zeros out `withdrawSharesOf[periods[i]][from]`, any unauthorized call to `claimWithdraw` by the blocklisted user will naturally calculate `assets = 0` and revert with `NoWithdrawalAmount(period)`. Removing the `isWithdrawClaimed` mutation guarantees that if the user is unblocklisted, they begin with a clean slate for future withdrawals.

```diff
   function rescueWithdrawFromBlocklisted(address from, address to, uint256[] calldata periods) external ... {
       for (uint256 i = 0; i < len; i++) {
           uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];
           if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
           if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
           if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);
          
           withdrawSharesOf[periods[i]][to] += _withdrawOf;
           withdrawSharesOf[periods[i]][from] = 0;
-           isWithdrawClaimed[periods[i]][from] = true;
           rescuedShares[i] = _withdrawOf;
       }
       emit WithdrawRescuedFromBlocklisted(from, to, periods, rescuedShares);
   }
```
