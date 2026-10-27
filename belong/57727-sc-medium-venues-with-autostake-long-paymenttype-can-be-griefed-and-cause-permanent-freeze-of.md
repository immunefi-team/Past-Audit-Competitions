# 57727 sc medium venues with autostake long paymenttype can be griefed and cause permanent freeze of long token

**Submitted on Oct 28th 2025 at 13:56:53 UTC by @iamephraim for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57727
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

The Staking contract allows attackers to permanently lock LONG token by exploiting the stake array growth mechanism of any user. When venues enable autostake payment type, attackers can make thousands of tiny deposits to the venue's address, causing the `stakes[venue]` array to grow exponentially. This creates a gas exhaustion attack where withdrawal operations fail due to O(n) complexity, permanently freezing the venue's LONG tokens.

### Vulnerability Details

The Staking contract contains an issue that allows attackers to permanently lock LONG token funds for venues using AutoStake functionality in its venue rules. The attack exploits the `_deposit` function's behavior of creating a new array entry for every deposit, regardless of size. An attacker can make thousands of tiny LONG token deposits `(1 wei each)` to a venue's address, causing the `stakes[venue]` array to grow at an exponential rate. This creates a gas exhaustion attack where any withdrawal attempt fails due to the O(n) complexity of `_consumeUnlockedSharesOrRevert` and `_removeAnySharesFor` functions. The victim becomes permanently locked out of their LONG token funds since even emergency withdrawals become gas-prohibitive. While venues are the primary target due to AutoStake venue rules, any user can be griefed through direct LONG token deposits.

The venue rules structure looks thus and available payment types:

```solidity
/// @title LongPaymentTypes
/// @notice Venue-allowed Long payment options.
enum LongPaymentTypes {
    NoType,
    AutoStake,
    AutoConvert
}

/// @title VenueRules
/// @notice Venue-level configuration for payment and bounty types.
struct VenueRules {
    PaymentTypes paymentType;
    BountyTypes bountyType;
    LongPaymentTypes longPaymentType;
}
```

Here are the functions involved for this attack path:

`deposit()` function which is made external from Solady's ERC4626 vault contract:

```solidity
function deposit(uint256 assets, address to) public virtual returns (uint256 shares) {
        if (assets > maxDeposit(to)) _revert(0xb3c61a83); // `DepositMoreThanMax()`.
        shares = previewDeposit(assets);
        _deposit(msg.sender, to, assets, shares);
    }
```

`_deposit()` function from the Belong's Stake contract that shows the `stakes[venue]` array and how attackers can grow the array exponentially with tiny LONG deposits:

```solidity
 function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
        super._deposit(by, to, assets, shares);
        // lock freshly minted shares
        stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
    }
```

`_consumeUnlockedSharesOrRevert()` function from the Belong's Stake contract that shows how a venue's `stakes[venue]` array is accessed and reduced:

```solidity
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

`_removeAnySharesFor()`:

```solidity
  /// @notice Removes shares from stake entries regardless of lock status (used in emergency flows).
    /// @dev Swap-and-pop for full consumption; partial consumption reduces the entry in-place.
    /// @param staker Address whose stake entries are modified.
    /// @param shares Number of shares to remove.
    function _removeAnySharesFor(address staker, uint256 shares) internal {
        Stake[] storage userStakes = stakes[staker];
        uint256 remaining = shares;

        for (uint256 i; i < userStakes.length && remaining > 0;) {
            uint256 stakeShares = userStakes[i].shares;
            if (stakeShares <= remaining) {
                remaining -= stakeShares;
                userStakes[i] = userStakes[userStakes.length - 1];
                userStakes.pop();
                // don't ++i: a new element is now at index i
            } else {
                userStakes[i].shares = stakeShares - remaining;
                remaining = 0;
                unchecked {
                    ++i;
                }
            }
        }
    }
```

## Impact Details

This attack affects all stakers in the contract as any address can be targeted for griefing, not just venues with AutoStake. Attackers can make tiny deposits to any staker's address, causing their stake arrays to grow and making withdrawals impossible due to gas exhaustion.

{% hint style="warning" %}
Recommendations (summary): implement minimum deposit thresholds, array size limits per user, stake consolidation mechanisms, and gas-efficient withdrawal algorithms to prevent array manipulation attacks.
{% endhint %}

## References

* The vulnerable `deposit` function\
  <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L245>
* Public function to get venue rules - `generalVenueInfo`\
  <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/BelongCheckIn.sol#L248>
* Line showing the flow of LONG token from BelongCheckIn to Stake contract\
  <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/BelongCheckIn.sol#L486-L490>

## Proof of Concept

{% stepper %}
{% step %}

### Target Selection

* Identify a venue that has AutoStake enabled in their payment rules.
* The venue's address will be the target for the griefing attack.
  {% endstep %}

{% step %}

### Array Population Attack

* Attacker makes thousands of tiny LONG token deposits (1 wei each) to the venue's address.
* Each deposit calls `_deposit()` which pushes a new Stake entry to `stakes[venue]`.
* The array grows from `n entries` to 10,000+ entries with minimal cost.
  {% endstep %}

{% step %}

### Gas Exhaustion in Normal Withdrawal

* Venue attempts to withdraw their staked LONG token funds normally. `_withdraw()` calls `_consumeUnlockedSharesOrRevert()` which loops through all array entries.
* The O(n) complexity causes gas exhaustion with 10,000+ entries.
* Transaction fails due to gas limit; LONG token funds remain locked.
  {% endstep %}

{% step %}

### Emergency Withdrawal Also Fails

* Venue attempts emergency withdrawal via `emergencyWithdraw()` or `emergencyRedeem()`.
* These functions call `_removeAnySharesFor()` which also loops through the entire array.
* Same gas exhaustion occurs; emergency withdrawal fails.
* Venue is completely locked out of their LONG token funds.
  {% endstep %}

{% step %}

### Permanent DoS Condition

* No mechanism exists to consolidate or limit stake entries.
* Array cannot be cleared or reduced in size by regular or emergency flows.
* Venue's LONG token funds are permanently inaccessible.
  {% endstep %}
  {% endstepper %}
