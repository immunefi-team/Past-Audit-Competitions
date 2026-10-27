# 49738 sc insight active users in prize pool loose invested raffle tickets when raffle removeprize is called&#x20;

**Submitted on Jul 18th 2025 at 22:19:54 UTC by @blackgrease for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49738
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

#### Note: The severity and impact mismatch is intentional. While the stated impact is *Contract fails to deliver promised returns, but doesn't lose value*, I believe the overall severity is High in relation to the broken invariant and the damage to users.

## Summary

An Admin calling `Raffle::removePrize`, causes all users in an active prize pool to loose all their raffle tickets due to the function not implementing logic to either stop execution if there are active users or to refund raffle tickets to users. This results in damage to the users and the protocol.

## Description

The `Raffle` contract allows users to join a prize pool using the raffle tickets they have earned from `Spin`. Prize pool creation and deactivation is handled by an Admin.

The Issue

When an active prize pool is made inactive by the admin calling the `Raffle::removePrize` function, there are no checks to see if there are any active users who have joined the raffle - for that particular prize pool. Once made inactive, users will loose their raffle tickets. A prize pool, once made inactive, cannot be made active.

This issue breaks one of the contract's invariants. "This \[Raffle] contract allows users to spend the raffle tickets they've earned from the Spin contract to enter drawings for prizes that can have multiple winners." — by causing users to incorrectly lose their raffle tickets while still being active participants, it neutralizes the purpose of earning prizes from drawings.

Furthermore, the `removePrize` function contains no logic to refund players their invested raffle tickets. This causes a financial impact to users. Playing the `Spin` game requires an investment of 2 Plume per each spin. In order to be the winner and earn the random raffle ticket reward, a large investment of time and Plume tokens is required.

Users will permanently lose the raffle tickets they have invested.

This issue can be triggered by:

* normal operations of the `Raffle` contract (i.e. admin calling `removePrize`)
* a malicious admin

Affected Code

Below is the problematic `Raffle::removePrize` function:

{% code title="Raffle.sol" %}

```solidity
function removePrize(uint256 prizeId) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) {
    prizes[prizeId].isActive = false;
    
    //@audit-high: There is no logic to refund or reemburse users in prize pool OR Logic to make sure no users have joined this prizePool 
    
    // Remove from prizeIds array
    uint256 len = prizeIds.length;
    for (uint256 i = 0; i < len; i++) { 
        if (prizeIds[i] == prizeId) {
            prizeIds[i] = prizeIds[len - 1];
            prizeIds.pop();
            break;
        }
    }     
    emit PrizeRemoved(prizeId);
}
```

{% endcode %}

## Impact

#### Note: The severity and impact mismatch is intentional. While the stated impact is *Contract fails to deliver promised returns, but doesn't lose value*, I believe due to the overall severity is High in relation to the impact on users and a broken invariant.

The assessed severity of this issue is considered High due to:

1. Contract invariant being broken. The invariant from the documentation, "This \[Raffle] contract allows users to spend the raffle tickets they've earned from the Spin contract to enter drawings for prizes that can have multiple winners." becomes broken as a user will no longer be able to spend the raffle tickets that they have earned.
2. Users will lose their investments used to obtain raffle tickets. Obtaining raffle tickets requires daily investment - 2 PLUME - on the `Spin` contract. As per the documentation, daily players get a higher streak count which multiples how many raffle tickets they will receive. Therefore, daily participation is rewarded. A user losing their raffle tickets will result in dissatisfaction and a negative outlook towards the protocol. This will cause a decrease in participation of the Spin-Raffle system.
3. The protocol will experience reputation damage due to the broken contract logic and the loss of user investments. This will further negatively impact the overall outlook of users towards the Plume ecosystem.
4. While this issue is triggered under normal usage, an admin - turned malicious - can trigger this issue to harm users.

Furthermore, the likelihood of occurring is High as this issue will be triggered under normal execution (i.e. admin calling `Raffle::removePrize` as per normal duties).

## Mitigation

The recommended mitigation to fix this issue has two approaches. Either prevent deactivation while users are active, or refund users when a prize pool is removed.

{% stepper %}
{% step %}

### Revert if active users

Check for active users in the prize pool and revert the call if any are present. This prevents deactivation while users have invested tickets.
{% endstep %}

{% step %}

### Refund tickets to players (Recommended)

When removing a prize pool, iterate through contributors and return their invested raffle tickets back to the `Spin` contract (or appropriate accounting) so users do not lose value.
{% endstep %}
{% endstepper %}

Either one of these fixes will protect users from incorrectly losing their raffle tickets after joining a prize pool.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/4cb5b1c5f8668aaad9a0891790bc64a9>

## Proof of Concept

As proof of the validity of this issue — while not required — a runnable PoC and a walk-through are provided in the private gist linked above. The walk-through outlines the workings of the script.

Run test with:

```
forge test --mt testUserLoosesRaffleTicketsOnRemovePrize --via-ir -vvv
```

Walk-through

{% stepper %}
{% step %}

### Admin creates a prize pool

Admin creates a prize Pool by calling `Raffle::addPrize()`.
{% endstep %}

{% step %}

### Players obtain raffle tickets

User1 and User2 who have been long term players in Spin have earned 20 raffle tickets each.
{% endstep %}

{% step %}

### Players join the prize pool

They join the prize pool with all their raffle tickets using `Raffle::spendRaffle()`.
{% endstep %}

{% step %}

### Admin removes the prize pool

For different reasons, Admin removes the prize pool therefore making it inactive.
{% endstep %}

{% step %}

### Prize pool becomes unusable

Once inactive, a prize pool cannot be made active and no winner can be selected from it.
{% endstep %}

{% step %}

### Users lose their raffle tickets

User1 and User2 have lost their raffle tickets without any reward and their investment in the Plume system (played with 2 Plume \* days\_spun = 20 raffle tickets).
{% endstep %}
{% endstepper %}

Attached is the output of the Foundry console in case there are any issues running the PoC.
