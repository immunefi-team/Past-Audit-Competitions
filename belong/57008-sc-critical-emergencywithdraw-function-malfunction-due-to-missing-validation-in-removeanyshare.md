# 57008 sc critical emergencywithdraw function malfunction due to missing validation in removeanysharesfor

**Submitted on Oct 22nd 2025 at 15:38:01 UTC by @Happy\_Hunter for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57008
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The Staking contract's `emergencyWithdraw()` function does not work properly when share transfers occur because the `_removeAnySharesFor()` function is missing the critical validation check:

if (remaining != 0) revert MinStakePeriodNotMet()

When users transfer their sLONG shares to another address, the stake records remain in the original owner's account, creating orphaned entries. The `emergencyWithdraw()` function then succeeds even when called on an address with no stake records, because `_removeAnySharesFor()` silently returns without validating whether it successfully removed the required shares. This causes the function to process withdrawals it should reject, leaving permanent orphaned stake records that corrupt state data used by third-party integrations.

### Vulnerability Details

The Staking contract has two internal functions for managing stake records:

1. `_consumeUnlockedSharesOrRevert()` - Used in normal withdrawals (SAFE):

```solidity
// Lines 258-290
function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 remaining = need;
    
    for (uint256 i; i < userStakes.length && remaining > 0;) {
        // ... consume unlocked shares ...
    }
    
    if (remaining != 0) revert MinStakePeriodNotMet();  // VALIDATES
}
```

2. `_removeAnySharesFor()` - Used in emergency withdrawals (VULNERABLE):

```solidity
// Lines 296-315
function _removeAnySharesFor(address staker, uint256 shares) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 remaining = shares;
    
    for (uint256 i; i < userStakes.length && remaining > 0;) {
        // ... remove any shares regardless of lock status ...
    }
    
    // MISSING: if (remaining != 0) revert MinStakePeriodNotMet();
    // Function completes successfully even if it couldn't remove all shares!
}
```

The `_removeAnySharesFor()` function is missing the validation check `if (remaining != 0) revert`. This means emergency withdrawals succeed even when the owner has insufficient or zero stake records, completely bypassing the lock mechanism.

{% stepper %}
{% step %}

### Initial Deposit

Alice deposits 1000 LONG using account\_01

* Receives 1000 sLONG shares
* `balanceOf(account_01)` = 1000
* `stakes[account_01][0]` = `{shares: 1000, timestamp: T}`
* Shares are locked for `minStakePeriod` (1 day by default)
  {% endstep %}

{% step %}

### Emergency Situation

Alice urgently needs her LONG tokens immediately but cannot wait for the lock period to expire. The normal `withdraw()` function will revert with `MinStakePeriodNotMet`.
{% endstep %}

{% step %}

### Share Transfer

Alice transfers all 1000 sLONG shares to account\_02 (also owned by her):

* `balanceOf(account_01)` = 0 (updated by ERC20 transfer)
* `balanceOf(account_02)` = 1000 (updated by ERC20 transfer)
* `stakes[account_01][0]` = `{shares: 1000, timestamp: T}` (NOT updated - remains orphaned)
* `stakes[account_02]` = `[]` (empty - no stake records transferred)
  {% endstep %}

{% step %}

### Approval

Alice approves account\_01 to spend account\_02's shares
{% endstep %}

{% step %}

### emergencyWithdraw Execution

Alice calls `emergencyWithdraw(1000, account_01, account_02)` from account\_01:

* `_removeAnySharesFor(account_02, 1000)` attempts to remove stake records from account\_02
* Loop through `stakes[account_02]` (empty array) - loop doesn't execute
* `remaining = 1000` (unchanged)
* Missing validation: function returns without checking `if (remaining != 0) revert MinStakePeriodNotMet()`
* `_burn(account_02, 1000)` succeeds (account\_02 has the share balance)
* Alice receives 900 LONG (1000 - 10% penalty)
  {% endstep %}

{% step %}

### Corrupt State

The withdrawal succeeds despite account\_02 having no stake records to validate:

* `stakes[account_01]` remains with orphaned record `{shares: 1000, timestamp: T}`
* `balanceOf(account_01)` = 0 (no actual shares)
* Third-party integrations reading `stakes[account_01]` see 1000 locked shares that don't exist
  {% endstep %}
  {% endstepper %}

### Impact Details

The `emergencyWithdraw()` function fails to properly validate stake ownership due to the missing `if (remaining != 0) revert` check in `_removeAnySharesFor()`. When shares are transferred between addresses, stake records remain orphaned in the original owner's account while the new owner has the actual share balance but no stake entries. The `_removeAnySharesFor()` function silently succeeds even when it cannot find any stake records to remove, allowing `emergencyWithdraw()` to process withdrawals for addresses that have no locked stakes. This breaks the function's intended behavior of only allowing emergency withdrawals for properly staked positions and creates permanent orphaned stake records that cannot be cleaned up.

If there is any third-party integration that uses the `stakes` variable to track staked positions, locked amounts, or time-lock status, this flaw is very serious as it provides completely incorrect data where addresses show locked stakes they don't own while actual share holders show no stake records at all.

### References

* `contracts/v2/periphery/Staking.sol`
* Solady ERC4626: <https://github.com/Vectorized/solady/blob/main/src/tokens/ERC4626.sol>

### Recommendation

Add the missing validation to `_removeAnySharesFor()` by mirroring the check in `_consumeUnlockedSharesOrRevert()`:

```solidity
function _removeAnySharesFor(address staker, uint256 shares) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 remaining = shares;
    
    for (uint256 i; i < userStakes.length && remaining > 0;) {
        uint256 stakeShares = userStakes[i].shares;
        if (stakeShares <= remaining) {
            remaining -= stakeShares;
            userStakes[i] = userStakes[userStakes.length - 1];
            userStakes.pop();
        } else {
            userStakes[i].shares = stakeShares - remaining;
            remaining = 0;
            unchecked {
                ++i;
            }
        }
    }
    
    // ADD THIS CHECK:
    if (remaining != 0) revert MinStakePeriodNotMet();
}
```

This enforces that emergency withdrawals can only proceed when the required shares are actually removed from the stake records, preserving the intended lock semantics and preventing orphaned stake entries.

## Link to Proof of Concept

<https://gist.github.com/SproutGoodHub/4b50713fe9e5396a279b064bd5116c01>

## Proof of Concept

### Test Code

```typescript
describe('emergencyWithdraw with Transferred Shares', () => {
  it('Should process withdrawal even when stake records remain in original owner account', async () => {
    const { staking, long, admin, user1, user2 } = await loadFixture(fixture);

    const amount = ethers.utils.parseEther('1000');

    // 1. User1 (account_01) deposits and gets shares locked
    await long.connect(admin).transfer(user1.address, amount);
    await long.connect(user1).approve(staking.address, amount);
    await staking.connect(user1).deposit(amount, user1.address);

    // Verify user1 has shares and they are locked
    expect(await staking.balanceOf(user1.address)).to.eq(amount);
    expect((await staking.stakes(user1.address, 0)).shares).to.eq(amount);

    // 2. User1 cannot do normal withdrawal due to time lock
    await expect(staking.connect(user1).withdraw(amount, user1.address, user1.address)).to.be.revertedWithCustomError(
      staking,
      'MinStakePeriodNotMet',
    );

    console.log(' User1 needs funds urgently but is locked...');

    // 3. User1 transfers all sLONG to User2 (account_02 - also owned by same person)
    await staking.connect(user1).transfer(user2.address, amount);

    // 4. User2 approves User1 to spend on their behalf
    await staking.connect(user2).approve(user1.address, amount);

    console.log(' User1 transferred shares to User2 and got approval');

    // 5. User1 calls emergencyWithdraw(amount, user1, user2)
    // BUG: _removeAnySharesFor() is missing the validation check
    // Normal withdraw uses: _consumeUnlockedSharesOrRevert() which has "if (remaining != 0) revert"
    // Emergency withdraw uses: _removeAnySharesFor() which has NO such check!
    //
    // This will:
    // - Call _removeAnySharesFor(user2, shares)
    //   - Loop through stakes[user2] (empty array)
    //   - remaining = shares (unchanged)
    //   - BUG: No validation! Function returns successfully despite remaining != 0
    // - Call _burn(user2, shares) - succeeds (user2 has balance)
    // - Transfer funds to user1 minus 10% penalty
    // - stakes[user1] remains untouched - ORPHANED RECORDS!
    const user1BalanceBefore = await long.balanceOf(user1.address);
    
    await staking.connect(user1).emergencyWithdraw(amount, user1.address, user2.address);

    const user1BalanceAfter = await long.balanceOf(user1.address);
    const penaltyAmount = amount.mul(1000).div(10000); // 10% penalty
    const expectedPayout = amount.sub(penaltyAmount);

    console.log(' emergencyWithdraw SUCCEEDED despite missing stake records!');
    console.log('  - _removeAnySharesFor() did NOT revert despite stakes[user2] being empty');
    console.log('  - Missing validation: if (remaining != 0) revert');
    console.log('  - Withdrawal processed: Penalty', ethers.utils.formatEther(penaltyAmount), 'LONG, Received', ethers.utils.formatEther(expectedPayout), 'LONG');

    // Verify the withdrawal succeeded
    expect(user1BalanceAfter.sub(user1BalanceBefore)).to.eq(expectedPayout);
    expect(await staking.balanceOf(user2.address)).to.eq(0);

    // CRITICAL BUG: stakes[user1] still exists with orphaned records!
    expect((await staking.stakes(user1.address, 0)).shares).to.eq(amount);

    console.log(' CRITICAL: Missing validation creates orphaned stake records:');
    console.log('  - stakes[user1][0].shares =', ethers.utils.formatEther(amount));
    console.log('  - balanceOf(user1) = 0');
    console.log('  - This breaks third-party integrations reading stakes[]!');
  });
});
```

### Running the Test

```bash
npm test -- test/v2/platform/staking.test.ts --grep "emergencyWithdraw"
```

### Test Output

```
  Staking
    emergencyWithdraw with Transferred Shares
 User1 needs funds urgently but is locked...
 User1 transferred shares to User2 and got approval
 emergencyWithdraw SUCCEEDED despite missing stake records!
  - _removeAnySharesFor() did NOT revert despite stakes[user2] being empty
  - Missing validation: if (remaining != 0) revert
  - Withdrawal processed: Penalty 100.0 LONG, Received 900.0 LONG
 CRITICAL: Missing validation creates orphaned stake records:
  - stakes[user1][0].shares = 1000.0
  - balanceOf(user1) = 0
  - This breaks third-party integrations reading stakes[]!
      ✓ Should process withdrawal even when stake records remain in original owner account

·----------------------------------|---------------------------|-------------|-----------------------------·
|       Solc version: 0.8.27       ·  Optimizer enabled: true  ·  Runs: 200  ·  Block limit: 30000000 gas  │
·································|···························|·············|····························
|  Methods                                                                                                 │
·············|·······················|·············|·············|·············|···············|··············
|  Contract  ·  Method             ·  Min        ·  Max        ·  Avg        ·  # calls      ·  usd (avg)  │
·············|·····················|·············|·············|·············|···············|··············
|  Staking   ·  deposit            ·     118480  ·     196480  ·     175115  ·            5  ·          -  │
·············|·····················|···············|·············|·············|···············|··············
|  Staking   ·  emergencyWithdraw  ·     114692  ·     134555  ·     123509  ·           10  ·          -  │
·············|·····················|···············|·············|·············|···············|··············
|  USDCMock  ·  approve            ·      53343  ·      53644  ·      53579  ·            5  ·          -  │
·············|·····················|···············|·············|·············|···············|··············
|  USDCMock  ·  transfer           ·      53635  ·      61114  ·      59613  ·            5  ·          -  │
·············|·····················|···············|·············|·············|···············|··············
|  Deployments                     ·                                         ·  % of limit   ·             │
···································|·············|·············|·············|···············|··············
|  LONG                            ·          -  ·          -  ·    1699924  ·        5.7 %  ·          -  │
···································|·············|·············|·············|···············|··············
|  Staking                         ·          -  ·          -  ·    1803176  ·          6 %  ·          -  │
·----------------------------------|-------------|-------------|-------------|---------------|-------------·

  4 passing (12s)
```
