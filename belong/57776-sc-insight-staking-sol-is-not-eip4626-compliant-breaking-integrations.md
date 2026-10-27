# 57776 sc insight staking sol is not eip4626 compliant breaking integrations

**Submitted on Oct 28th 2025 at 20:44:29 UTC by @Another for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57776
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The Staking contract fails to properly override ERC-4626's `maxWithdraw` and `maxRedeem` functions to consider the `minStakePeriod` parameter. This violates the ERC-4626 standard specification, which requires these functions to return the actual maximum amounts that can be withdrawn or redeemed without causing a revert.

## Vulnerability Details

The current implementation inherits the base Solady ERC-4626 `maxWithdraw` and `maxRedeem` functions, which return the user's full balance without considering the staking lock period. However, the `_withdraw` function enforces `_consumeUnlockedSharesOrRevert`, which will revert if the minimum staking period hasn't been met. This creates a discrepancy where the `maxWithdraw` and `maxRedeem` functions indicate more is available than can actually be withdrawn.

Excerpt from the inherited implementations:

```solidity
    /// @dev Returns the maximum amount of the underlying asset that can be withdrawn
    /// from the `owner`'s balance in the Vault, via a withdraw call.
    ///
    /// - MUST return a limited value if `owner` is subject to some withdrawal limit or timelock.
    /// - MUST NOT revert.
    function maxWithdraw(address owner) public view virtual returns (uint256 maxAssets) {
        maxAssets = convertToAssets(balanceOf(owner));
    }

    /// @dev Returns the maximum amount of Vault shares that can be redeemed
    /// from the `owner`'s balance in the Vault, via a redeem call.
    ///
    /// - MUST return a limited value if `owner` is subject to some withdrawal limit or timelock.
    /// - MUST return `balanceOf(owner)` otherwise.
    /// - MUST NOT revert.
    function maxRedeem(address owner) public view virtual returns (uint256 maxShares) {
        maxShares = balanceOf(owner);
    }
```

According to the ERC-4626 standard:

* maxWithdraw: MUST factor in both global and user-specific limits, like if withdrawals are entirely disabled (even temporarily) it MUST return 0.
* maxRedeem: MUST factor in both global and user-specific limits, like redemption being disabled; MUST NOT revert.

The contract's `_withdraw` calls `_consumeUnlockedSharesOrRevert`, which reverts if the minimum staking period isn't met, meaning calls to `maxWithdraw` / `maxRedeem` can return values that will cause a subsequent `withdraw`/`redeem` to revert.

Excerpt from `_withdraw` / stake bookkeeping:

```solidity
    /// @dev Gas-efficient withdrawal with single pass consumption of unlocked shares.
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

        if (remaining != 0) revert MinStakePeriodNotMet();
    }
```

## Impact Details

ERC-4626 non-compliance leads to integration failures with other protocols and aggregators expecting ERC-4626 compliance; they may malfunction, breaking composability.

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L248-L290>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/node\\_modules/solady/src/tokens/ERC4626.sol#L342-L360>
* <https://eips.ethereum.org/EIPS/eip-4626>

## Proof of Concept

The following test demonstrates the discrepancy: `maxWithdraw` returns the full balance, but `withdraw` reverts due to `minStakePeriod` not being met.

```ts
describe('ERC4626 Compliance - `maxWithdraw``', () => {
  it('maxWithdraw returns full balance but withdraw reverts due to minStakePeriod breaking ERC4626', async () => {
    const { staking, long, admin, user1 } = await loadFixture(fixture);

    const amount = ethers.utils.parseEther('1000');

    // User1 deposits
    await long.connect(admin).transfer(user1.address, amount);
    await long.connect(user1).approve(staking.address, amount);
    await staking.connect(user1).deposit(amount, user1.address);

    // Check maxWithdraw - should be the full amount (showing non-compliance)
    const maxWithdraw = await staking.connect(user1).maxWithdraw(user1.address);
    expect(maxWithdraw).to.eq(amount);
    expect(maxWithdraw).to.not.eq(0);

    // Attempt to withdraw the max amount - should revert because `minStakePeriod` not met
    await expect(staking.connect(user1).withdraw(maxWithdraw, user1.address, user1.address))
      .to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
  });
});
```
