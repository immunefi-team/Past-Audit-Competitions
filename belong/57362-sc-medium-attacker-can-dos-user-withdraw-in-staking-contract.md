# 57362 sc medium attacker can dos user withdraw in staking contract

**Submitted on Oct 25th 2025 at 14:14:19 UTC by @iehnnkta for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57362
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

`Staking::deposit()` is used to deposit Long token for `sLONG` tokens. The deposited assets can only be withdrawn after a certain period. The receiver can withdraw assets by looping through the `stakes[to]` array.

### Vulnerability Details

There is no proper validation on the number of deposits that can be done to a user. A malicious actor can deposit a tiny amount many times (e.g., 1 wei), creating thousands of entries in the `stakes[to]` array. When the receiver tries to redeem or withdraw, the contract calls `_withdraw`, which uses `_consumeUnlockedSharesOrRevert` and iterates over `stakes[to]`:

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

If `userStakes.length` grows very large (for example, > 20,000), iterating through the array can exceed the block gas limit and make withdraw/redeem impossible for the recipient.

### Impact Details

* Attacker cost is minimal (tiny deposits, low BSC gas).
* Recipient may be unable to withdraw funds because the loop over many tiny deposits exceeds the block gas limit.
* Funds could be effectively frozen for the recipient (permanent freezing unless mitigations are added).

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L242-L246>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L258-L290>

## Proof of Concept

{% stepper %}
{% step %}
Alice deposits 100e18 long tokens to Bob.
{% endstep %}

{% step %}
Attacker sees this transaction in mempool and front-runs with many tiny deposits (e.g., 1 wei) to Bob, repeated hundreds or thousands of times.
{% endstep %}

{% step %}
This enormously increases `stakes[bob]` array length.
{% endstep %}

{% step %}
Alice's original transaction executes (Bob now has the large deposit plus many tiny stakes).
{% endstep %}

{% step %}
After the stake lock period elapses, Bob attempts to withdraw the deposited 100e18 tokens.
{% endstep %}

{% step %}
The withdraw/redeem call iterates through `stakes[bob]`. Because of the huge number of tiny deposits injected by the attacker, the transaction consumes excessive gas.
{% endstep %}

{% step %}
Worst case: attacker injects enough tiny deposits that the withdraw transaction would require more gas than a block allows. In that case, Bob cannot withdraw the funds at all.
{% endstep %}
{% endstepper %}

## Suggested Mitigations (not exhaustive)

* Limit number of stakes per address (reject deposits if the user's stake array is above a threshold).
* Aggregate deposits for a given recipient within a short time window instead of pushing a new stake array entry per deposit.
* Use a different data structure enabling O(1) or bounded-cost processing per deposit (e.g., mapping of timestamp buckets or linked lists with per-call gas-limited processing).
* Add a mechanism allowing the recipient to process stakes in multiple transactions (pagination) or allow an emergency admin to compact stakes.
