# 57799 sc low retroactive lock period changes affect existing stakes

**Submitted on Oct 28th 2025 at 23:16:49 UTC by @Another for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57799
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

Brief/Intro

The `setMinStakePeriod` function allows the contract owner to change the minimum stake period, which retroactively affects all existing stakes.

## Vulnerability Details

The `setMinStakePeriod` function allows the contract owner to change the minimum stake period:

{% code title="Staking.sol — setMinStakePeriod" %}

```solidity
    /// @dev Reverts if `period == 0`.
    /// @param period New minimum stake period in seconds.
    function setMinStakePeriod(uint256 period) external onlyOwner {
        require(period > 0, MinStakePeriodShouldBeGreaterThanZero());
        minStakePeriod = period;/
        emit MinStakePeriodSet(period);
    }
```

{% endcode %}

Each stake recorded in the `deposit` function only sets a `timestamp` without storing the `minStakePeriod` that was in effect at deposit time. During withdrawal, the `_consumeUnlockedSharesOrRevert` function uses the current global `minStakePeriod` for all stake validation:

{% code title="Staking.sol — deposit & withdraw hooks" %}

```solidity
    function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
        super._deposit(by, to, assets, shares);
        // lock freshly minted shares
        stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
    }

    /// @dev Gas-efficient withdrawal with single pass consumption of unlocked shares.
    function _withdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal override {
        _consumeUnlockedSharesOrRevert(_owner, shares);
        super._withdraw(by, to, _owner, assets, shares);
    }
```

{% endcode %}

{% code title="Staking.sol — \_consumeUnlockedSharesOrRevert" %}

```solidity
    // ============================== Stake Bookkeeping ==============================

    /// @notice Consumes exactly `need` unlocked shares or reverts.
    /// @dev Single pass; swap-and-pop removal; partial consumption in-place.
    function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
        Stake[] storage userStakes = stakes[staker];
        uint256 _min = minStakePeriod;
        uint256 nowTs = block.timestamp;
        uint256 remaining = need;

        for (uint256 i; i < userStakes.length && remaining > 0;) {
            Stake memory s = userStakes[i];
            if (nowTs >= s.timestamp + _min) {
                uint256 take = s.shares <= remaining ? s.shares : remaining;
                if (take == s.shares) {
                    // full consume → swap and pop
                    remaining -= take;
                    userStakes[i] = userStakes[userStakes.length - 1];
                    userStakes.pop();
                    // don't ++i: a new element is now at index i
                } else {
                    // partial consume
                    userStakes[i].shares = s.shares - take;
                    remaining = 0;
                    unchecked {
                        ++i;
                    }
                }
            } else {
                unchecked {
                    ++i;
                }
            }
        }

        if (remaining != 0) revert MinStakePeriodNotMet();
    }
```

{% endcode %}

Because the contract uses a single global `minStakePeriod` for validation, changing `minStakePeriod` retroactively affects all existing stakes and can either shorten or extend their effective lock times.

Recommendation: store an explicit end timestamp (unlock time) for each stake at deposit time, and validate against that stored value when withdrawing. This prevents retroactive changes to lock lengths for already-created stakes.

## Impact Details

If the `minStakePeriod` is increased after a user deposited, users who would otherwise be able to withdraw their shares can be prevented from withdrawing until the new (longer) period elapses — effectively extending the lock on already existing stakes.

## Proof of Concept

<details>

<summary>Test demonstrating that increasing minStakePeriod blocks withdrawal for existing stakes</summary>

Add the following test to staking.test.ts

```ts
describe('Minimum Stake Period Changes Affect Existing Stakes', () => {
  it('should prevent withdrawal when minStakePeriod is increased after deposit', async () => {
    const { staking, long, admin, user1 } = await loadFixture(fixture);

    const amount = ethers.utils.parseEther('1000');

    // User deposits with initial 1 day lock period
    await long.connect(admin).transfer(user1.address, amount);
    await long.connect(user1).approve(staking.address, amount);
    await staking.connect(user1).deposit(amount, user1.address);

    // Wait 12 hours (half the original lock period)
    await ethers.provider.send('evm_increaseTime', [23 * 60 * 60]);
    await ethers.provider.send('evm_mine', []);

    // Owner increases to minStakePeriod to 2 days
    await staking.connect(admin).setMinStakePeriod(30 * 24 * 60 * 60);

    // Try to withdraw - should fail even though original period was almost over
    await expect(staking.connect(user1).withdraw(amount, user1.address, user1.address))
      .to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');

    // Wait another 6 hours (total 24 hours - original period would be complete)
    await ethers.provider.send('evm_increaseTime', [6 * 60 * 60]);
    await ethers.provider.send('evm_mine', []);

    // Still cannot withdraw because new 2-day period applies
    await expect(staking.connect(user1).withdraw(amount, user1.address, user1.address))
      .to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
  });
});
```

</details>

## References

<details>

<summary>Source lines referenced</summary>

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L128-L134>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L245>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L260>

</details>
