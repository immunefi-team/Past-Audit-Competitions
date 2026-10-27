# 49705 sc medium two vectors for unbounded gas consumption due to the normal raffle operations

* Submitted on Jul 18th 2025 at 15:49:15 UTC by @blackgrease for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #49705
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* Impacts:
  * Unbounded gas consumption

## Description

### Note

This report outlines a long term issue that is based on the current design of the `Raffle` contract.

## Summary

The `Raffle` contract has two griefing vectors affecting users and admin roles. This is due to how the contract handles inactive and active prizes forcing `removePrize` and `getPrizeDetails` to iterate over a large array potentially resulting in a denial of service where no prize can be removed and no prize details can be retrieved.

## Description

The `Raffle` contract allows users to use their obtained raffles from the `Spin` rewards system in order to obtain prizes. Prizes are set by an admin.

The Issue:

{% stepper %}
{% step %}

### How prize array grows

* Prizes are added by an admin by invoking `Raffle::addPrize(args)`. Each new prize increases the size of the `prizeIds` array.
  {% endstep %}

{% step %}

### How removal is restricted

* The only way to decrease the `prizeIds` array size is for the admin to call `Raffle::removePrize`. However, this can only be called on prizes that are active due to the modifier `prizeIsActive`. Prizes that have already been claimed cannot be removed.
  {% endstep %}

{% step %}

### Prize lifecycle

* When all winners have claimed a prize, the prize is set as Inactive in `Raffle::handleWinnerSelection`.
  {% endstep %}

{% step %}

### Inactive prizes remain in array

* The inactive prize is not removed from the `prizeIds` array.
  {% endstep %}
  {% endstepper %}

This means the `prizeIds` array will hold both active and inactive prizes. Over time, this array will grow in size and there is no limit set for its size.

Impacting functions:

* removePrize loops over the `prizeIds` array to find and remove an id.
* getPrizeDetails loops over the full `prizeIds` array and returns details for each entry.

Example: removePrize implementation in the contract

{% code title="removePrize(uint256 prizeId) — relevant excerpt" %}

```solidity
function removePrize(uint256 prizeId) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) {
    prizes[prizeId].isActive = false;
    
    // Remove from prizeIds array
    uint256 len = prizeIds.length;
    for (uint256 i = 0; i < len; i++) { //@audit: unbounded gas consumption
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

Example: getPrizeDetails implementation in the contract

{% code title="getPrizeDetails() — relevant excerpt" %}

```
```

{% endcode %}

```solidity
function getPrizeDetails() external view returns (PrizeWithTickets[] memory) {
    uint256 prizeCount = prizeIds.length;
    PrizeWithTickets[] memory prizeArray = new PrizeWithTickets[](prizeCount);
    
    for (uint256 i = 0; i < prizeCount; i++) { //@audit: unbounded gas consumption
        uint256 currentPrizeId = prizeIds[i];
        Prize storage currentPrize = prizes[currentPrizeId];
        
        prizeArray[i] = PrizeWithTickets({
            name: currentPrize.name,
            description: currentPrize.description,
            value: currentPrize.value,
            endTimestamp: currentPrize.endTimestamp,
            isActive: currentPrize.isActive,
            quantity: currentPrize.quantity,
            winnersDrawn: winnersDrawn[currentPrizeId],
            totalTickets: totalTickets[currentPrizeId],
            totalUsers: totalUniqueUsers[currentPrizeId]
        });
    }
    
    return prizeArray;
}
```

Alternatively, for both vectors, even if out-of-gas errors are not thrown, the high gas cost may make the functions infeasible to interact with, reducing their purpose/functionality.

## Impact

Unbounded gas consumption resulting in a Denial of Service, either by:

* out of gas
* or infeasibility to invoke due to high gas costs

Affected parties:

1. Admin: prevents incorrect or undesirable prizes from being removed from the `Raffle` contract.
2. Users: prevents them from identifying what prize to spend their raffle reward on, making them blind to the current prizes in the contract.

As a side effect, users not being able to get prize details may reduce participation in the gamified system and hinder user engagement for the Plume system.

## Mitigation

Recommended mitigations:

* Once a prize has been fully claimed by winners, the prize should be removed from the `prizeIds` array. Because events are emitted (e.g., when a prize is fully claimed), historical prize data remains accessible on-chain via explorers.
* Additionally, enforce a maximum limit to the number of active prizes which will bound the `prizeIds` array.

These mitigations ensure only active prizes remain in `prizeIds`, preventing unbounded loops and improving efficiency. For users, returned information will be limited to useful (active) prizes.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/2682d84574dcc145ddcb53524a9e3c19>

## Proof of Concept

As proof of the validity of this issue the reporter included a runnable PoC alongside a walk-through. The runnable script displays the gas consumption for both vectors.

Private Gist link: "<https://gist.github.com/blackgrease/2682d84574dcc145ddcb53524a9e3c19>"

Run both tests with:

```bash
forge test --mt testUnboundedGasConsumption -vvv --via-ir
```

The reporter also attached a screenshot of the Foundry console after the test displaying the gas consumption across both vectors.

<details>

<summary>Walk-through: Vector #1 (Admin)</summary>

StepAdmin adds prizes to the system as per duties. The prizeIds array grows with time.StepThe system works as normal. Prizes are claimed and made inactive. There are no prizes removed from the system by the admin during this time.StepThe prizeIds array contains active and inactive prizes.StepAn incorrect prize is submitted by the admin and is not active.StepThe admin attempts to remove the prize by invoking Raffle::removePrize(id).StepIf the incorrect prize is at the end of the prizeIds array the function must loop over all items to find it and may run out of gas before completing (or be too expensive). As a result, the incorrect prize can no longer be removed from the system and users will keep interacting with it.

</details>

<details>

<summary>Walk-through: Vector #2 (User)</summary>

StepAdmin adds prizes to the system as per duties. The prizeIds array grows with time.StepThe system works as normal. Prizes are claimed and made inactive. There are no prizes removed from the system by the admin during this time.StepThe prizeIds array contains active and inactive prizes.StepA user who has obtained raffle rewards from Spin wants to know which prize pool to participate in and calls getPrizeDetails.StepThe function loops over the entire array to load prize details.StepDue to the array's size, the function may run out of gas and the user fails to retrieve prize details (or it becomes infeasible due to gas costs).

</details>
