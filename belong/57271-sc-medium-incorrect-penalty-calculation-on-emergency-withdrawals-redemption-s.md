# 57271 sc medium incorrect penalty calculation on emergency withdrawals redemption s&#x20;

**Submitted on Oct 24th 2025 at 21:06:28 UTC by @blackgrease for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57271
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

Affected File: `Staking.sol`

The `Staking` contract has emergency withdrawal functions — `emergencyWithdraw` and `emergencyRedeem` — that allow a user to bypass the minimum staking period in order to remove their locked stakes. Stakes that have already been unlocked by passing the minimum staking period are eligible to be withdrawn normally by calling `Staking::withdraw` or `Staking::redeem`.

When emergency withdrawals are used, the removed stake has a penalty fee applied to it. However, the application of this penalty fee is incorrect. Both withdrawal related functions call the internal function `_emergencyWithdraw` which handles the penalty calculation.

The purpose of emergency withdrawals is to allow users to withdraw/redeem stakes that are still locked within the minimum staking period for emergency reasons. However, when a user calls this function, the penalty is incorrectly applied to all locked and unlocked assets, instead of only locked assets.

{% stepper %}
{% step %}

### Example scenario

* The Staking has a minimum staking period of 1 day and a 10% penalty fee
* A user stakes 200 LONG for 7 days; effectively completing the minimum staking period.
* They later deposit another 200 LONG.
* However, for emergency reasons they need to withdraw 400 LONG.
* As the initial 200 LONG has already passed the minimum staking period, these tokens should not be part of the penalty fee. The fee should only apply to the 200 LONG still locked.
* However, the code incorrectly applies the 10% penalty fee to the total 400 LONG, griefing the user even though they followed the initial staking flow.
* The user receives 360 LONG instead of 380 LONG.
  {% endstep %}
  {% endstepper %}

### The Problematic code

```solidity
//@audit: penalty incorrectly applied even when the token has already been unlocked. Should only be applied to tokens that are still locked.
function _emergencyWithdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal {
        require(shares > 0, SharesEqZero());

        uint256 penalty = FixedPointMathLib.fullMulDiv(assets, penaltyPercentage, SCALING_FACTOR);
        uint256 payout;
        unchecked {
            payout = assets - penalty; //@audit-issue: penalty is only for locked tokens. It does not check the amount of locked/unlocked tokens before applying
        }

        if (by != _owner) _spendAllowance(_owner, by, shares); //@audit-info: protects against other users stealing funds

        _removeAnySharesFor(_owner, shares);
        _burn(_owner, shares);

        LONG.safeTransfer(to, payout);
        LONG.safeTransfer(treasury, penalty);

        emit EmergencyWithdraw(by, to, _owner, assets, shares); //@audit-low: emits the wring data. should emit payout: for both assets and shares. shouldnt count input -> incorrect information
        // also emit standard ERC4626 Withdraw for indexers/analytics
        emit Withdraw(by, to, _owner, assets, shares);
    }
```

### Code Flow

* Withdrawals: `emergencyWithdraw` -> `_emergencyWithdraw` -> `_removeAnySharesFor` -> transfers happen
* Redemptions: `emergencyRedeem` -> `_emergencyWithdraw` -> `_removeAnySharesFor` -> transfers happen

## Impact

The impact of this issue is a griefing issue due to the Staking vault miscalculating withdrawal penalties leading to users being overcharged penalty fees. In line with the purpose of emergency withdrawals, only locked tokens should have penalties applied to them as they are bypassing the minimum staking period.

The amount of loss a staker incurs increases with higher penalties.

## Mitigation

The recommended mitigation is to have the `_removeAnySharesFor` only apply penalties to stakes that are still locked.

It is important to note that the current contract logic has the capacity to enforce checks on lock status of stakes due to staking operations having their timestamp stored in the `Stake` struct. Example from the contract:

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
        super._deposit(by, to, assets, shares);
        // lock freshly minted shares
        stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
    }
```

## Link to Proof of Concept

<https://gist.github.com/blackgrease/3656507e38d00baa6ec6f23b84171cac>

## Proof of Concept

A runnable Foundry PoC is provided in the gist linked above. The test simulates the described scenario.

* Run with:

```bash
forge test --mt testIncorrectPenaltyCalculationOnEmergencyWithdrawals --via-ir -vvv
```

Additionally, the gist contains a .txt Foundry stack trace for the test ("PoC\_StackTrace\_testIncorrectPenaltyCalculationOnEmergencyWithdrawals.txt").

Console Messages from the PoC:

```
Logs:
  --INCORRECT APPLICATION OF PENALTY ON EMERGENCY WITHDRAWALS--
  Initial staked amount:  1000000000000000000000
  Second staked amount after minimum staking period has passed:  1000000000000000000000
  Stake eligible to be unlocked:  1000000000000000000000
  Stake still locked:  1000000000000000000000
  Amount that should be received on emergency withdraw:  1900000000000000000000
  Actual amount received after emergency withdraw:  1800000000000000000000
  Loss To staker:  100000000000000000000
```

{% stepper %}
{% step %}

### Foundry setup (high-level)

1. Clone the Github repo:
   * `git clone https://github.com/belongnet/checkin-contracts.git`
2. Install the Foundry dependencies:
   * `forge install OpenZeppelin/openzeppelin-contracts-upgradeable@v5.4.0 --no-commit`
   * `forge install OpenZeppelin/openzeppelin-contracts@v5.4.0 --no-commit`
   * `npm install solady --force`
3. Update remappings in `foundry.toml` (example provided in the original report).
   {% endstep %}
   {% endstepper %}
