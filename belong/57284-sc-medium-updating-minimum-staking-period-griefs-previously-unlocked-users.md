# 57284 sc medium updating minimum staking period griefs previously unlocked users

**Submitted on Oct 24th 2025 at 23:18:14 UTC by @blackgrease for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57284
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

**Affected Files**: `Staking.sol`

The `Staking` contract requires users to stake their LONG tokens for a set minimum period. By default this is set to 1 day at the time of calling `Staking::initialize`.

The owner can update the minimum staking period. However, the current implementation applies the new minimum period to all stakes — including previously unlocked stakes — which can grief users by forcing already-unlocked stakes to become locked again for another minimum period.

Example scenario:

* Staker A stakes 1000 LONG for 2 days.
* The minimum staking period is 1 day, so after 1 day their stake becomes withdrawable.
* The owner updates the minimum staking period to 5 days (applies to all stakes).
* Staker A must now wait an additional 3 days to withdraw, despite already being eligible before the change.
* If an emergency occurs, the user must use emergency withdrawal and incur a penalty.

Note: If the stake age already exceeds the new minimum (e.g., 6 days), the stake is unaffected.

## Impact

This logic issue allows the owner to extend lockups retroactively, griefing users by restricting access to their tokens and forcing costly emergency withdrawals. Likelihood is High; impact depends on stake age and change magnitude.

## Mitigation

Recommended mitigation: change `setMinStakePeriod` semantics so it only affects stakes made after the change. For example, record the time of the min-period update and use per-stake deposit timestamps (already stored on stake) to check eligibility against the min-period in effect at deposit time. This ensures existing and already-unlocked stakes remain withdrawable.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/25f2d64e1c7cb5b0249a35be2d411934>

## Proof of Concept

A runnable Foundry PoC is available at the gist above.

Run with:

```bash
forge test --mt testIfChaningMinStakingGriefsUsers -vvv --via-ir
```

The gist also contains a `.txt` Foundry stack trace for the test: "PoC\_StackTrace-testIfChaningMinStakingGriefsUsers.txt".

{% stepper %}
{% step %}

### PoC description — test steps

* Note: The minimum staking period starts as 1 day.
* Test sequence:
  * Stake an amount into the `Staking` contract and confirm that while staking time < 1 day, withdrawals cannot be executed.
  * Advance time past the minimum staking period and confirm the funds become eligible for withdrawal.
  * Withdraw a small amount to confirm eligibility.
  * Update the minimum staking period.
  * Confirm that the previously unlocked tokens are locked again, preventing withdrawals and griefing the user.
    {% endstep %}
    {% endstepper %}

## Foundry Setup

{% stepper %}
{% step %}

### Clone the repo

git clone <https://github.com/belongnet/checkin-contracts.git>
{% endstep %}

{% step %}

### Install Foundry dependencies

1. forge install OpenZeppelin/openzeppelin-contracts-upgradeable\@v5.4.0 --no-commit
2. forge install OpenZeppelin/openzeppelin-contracts\@v5.4.0 --no-commit
3. npm install solady --force
   {% endstep %}

{% step %}

### Update remappings in foundry.toml

Replace or update the remappings to:

```toml
## Foundry Remappings
remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts","@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts"]
```

(If you previously had other remappings commented out due to npm issues, follow the file guidance in the PoC.)
{% endstep %}
{% endstepper %}

## Additional notes

* The issue is purely logical (permissioned owner action changing protocol behavior) and does not imply direct loss of token value, but it affects user experience and trust.
* The recommended fix is to ensure changes to the minimum staking period only apply to new stakes (or stakes created after the change) by referencing deposit time and the change timestamp in the withdrawal eligibility check.
