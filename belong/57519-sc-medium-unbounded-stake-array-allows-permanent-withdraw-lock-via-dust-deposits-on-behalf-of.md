# 57519 sc medium unbounded stake array allows permanent withdraw lock via dust deposits on behalf of victims

**Submitted on Oct 26th 2025 at 22:06:52 UTC by @v0id for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57519
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

The Staking contract allows anyone to call `deposit()` specifying an arbitrary `receiver` address. This is built on top of an ERC4626 vault:

Reference from OpenZeppelin ERC4626.deposit:

```solidity
function deposit(uint256 assets, address receiver) public virtual returns (uint256) {
    uint256 maxAssets = maxDeposit(receiver);
    if (assets > maxAssets) {
        revert ERC4626ExceededMaxDeposit(receiver, assets, maxAssets);
    }

    uint256 shares = previewDeposit(assets);
    _deposit(_msgSender(), receiver, assets, shares);

    return shares;
}
```

Staking contract overrides `_deposit` and appends a stake entry for `receiver`:

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
    super._deposit(by, to, assets, shares);
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

The staking feature implements a lock based on stake entries but has no minimum deposit amount required to create a locked stake. Because anyone can deposit on behalf of a victim with an arbitrary receiver address, an attacker can create many dust (very small) locked stakes for that victim. Withdraw and emergency withdraw iterate over the staker's stake array to consume unlocked shares:

Excerpt of `_withdraw` and the stake consumption loop:

```solidity
function _withdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal override {
    _consumeUnlockedSharesOrRevert(_owner, shares);
    super._withdraw(by, to, _owner, assets, shares);
}

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
```

The same pattern exists in `emergencyWithdraw` (another loop over the stakes).

Because there is no minimum stake size and no limit on the number of stakes per receiver, an attacker can create many tiny stakes (e.g., 1 wei) for a victim. When the victim later attempts to withdraw, the withdraw function must iterate over all stake entries; with sufficiently many entries the transaction can hit the EVM gas limit and revert, preventing the victim from withdrawing unlocked funds.

If the contract is behind a proxy this results in temporary freezing of funds (until some mitigation or upgrade). If not behind a proxy, the freezing can be permanent.

Suggested mitigation approaches mentioned:

* Require a minimum deposit amount for creating a locked stake.
* Prevent others from depositing on behalf of a user (change how the ERC4626 vault handles `deposit` to disallow deposits with arbitrary `receiver` on behalf of another user).
* Other fixes that limit or change stake bookkeeping to avoid unbounded arrays per user.

## Proof of Concept

{% stepper %}
{% step %}

### Create a dust locked stake for the victim

Call `deposit` on the ERC4626 vault with:

* receiver: the victim address
* amount: 1 wei
  {% endstep %}

{% step %}

### Repeat many times

Repeat the above deposit many times (e.g., \~2000 times) to create many stake entries for the victim.
{% endstep %}

{% step %}

### Victim attempts withdraw

The victim tries to call `withdraw` or `emergencyWithdraw`. The withdraw function iterates over the stake entries and the transaction runs out of gas (reverts), preventing withdrawal.
{% endstep %}
{% endstepper %}

## Impact

* Permanent or temporary freezing of funds for affected users (depending on whether the contract is upgradable behind a proxy).
