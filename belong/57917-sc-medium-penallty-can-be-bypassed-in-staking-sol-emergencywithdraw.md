# 57917 sc medium penallty can be bypassed in staking sol emergencywithdraw&#x20;

**Submitted on Oct 29th 2025 at 13:03:25 UTC by @Bx4 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57917
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * The contract loses the penalty funds too

## Description

### Brief/Intro

When a user invokes an emergency withdrawal they can bypass fees by withdrawing small amounts continuously.

### Vulnerability Details

In `Staking.sol` there is an internal function called `_emergencyWithdraw` which external functions such as `emergencyRedeem` and `emergencyWithdraw` call to make instant withdrawals. Emergency withdrawals attract a penalty. However, attackers can bypass this penalty by repeatedly withdrawing small amounts until they reach the desired total, avoiding the intended penalty on the full amount.

Example: If penalty is set to 1000 BPS (10%), an attacker can withdraw 10 assets 100 times to obtain 1000 assets while skipping the 10% penalty that would apply if withdrawing 1000 at once.

### Impact Details

The contract fails to retain or forward the intended penalty amount when an emergency withdrawal is invoked repeatedly by an attacker.

## References

* <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L201>
* <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L172>
* <https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/periphery/Staking.sol#L186>

## Proof of Concept

{% stepper %}
{% step %}

### Modify test amount

Edit the test in `staking.ts` and change the withdrawn `amount` to `10` (instead of `1000`).
{% endstep %}

{% step %}

### Observe penalty behavior

Run the test and observe that after execution the `penalty` equals `0`, demonstrating the penalty bypass when withdrawing small amounts repeatedly.
{% endstep %}
{% endstepper %}

Code excerpt of the test for context:

{% code title="staking.ts (excerpt)" %}

```
```

{% endcode %}

\`\`\`solidity it('emergencyWithdraw()', async () => { const { staking, long, treasury, admin, user1 } = await loadFixture(fixture);

```
  const amount = ethers.utils.parseEther('1000');

  await long.connect(admin).transfer(user1.address, amount);
  await long.connect(user1).approve(staking.address, amount);

  await staking.connect(user1).deposit(amount, user1.address);

  await expect(
    staking.connect(user1).emergencyWithdraw(amount.add(1), user1.address, user1.address),
  ).to.be.revertedWithCustomError(staking, 'WithdrawMoreThanMax');

  const tx = await staking.connect(user1).emergencyWithdraw(amount, user1.address, user1.address);

  const penalty = getPercentage(amount, await staking.penaltyPercentage());
  const payout = amount.sub(penalty);

  await expect(tx)
    .to.emit(staking, 'EmergencyWithdraw')
    .withArgs(user1.address, user1.address, user1.address, amount, amount);
  await expect(tx)
    .to.emit(staking, 'Withdraw')
    .withArgs(user1.address, user1.address, user1.address, amount, amount);
  expect(await staking.balanceOf(user1.address)).to.eq(0);
  expect(await long.balanceOf(staking.address)).to.eq(0);
  expect(await long.balanceOf(treasury.address)).to.eq(penalty);
  expect(await long.balanceOf(user1.address)).to.eq(payout);
  await expect(staking.stakes(user1.address, 0)).to.be.reverted;
});
```

```
```
