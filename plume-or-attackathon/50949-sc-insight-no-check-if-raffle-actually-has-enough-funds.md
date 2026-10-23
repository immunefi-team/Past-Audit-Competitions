# 50949 sc insight no check if raffle actually has enough funds

**Submitted on Jul 29th 2025 at 22:19:06 UTC by @PotEater for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50949
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

In the `Raffle.sol` contract, in the function `addPrize`, there is no check that the raffle actually has enough funds to cover the prize.

### Vulnerability Details

The function `addPrize` is used to add a new prize to the raffle system. However, there is no check that the contract actually has enough to cover the prize and can afford to offer such a prize.

Code snippet:

{% code title="Raffle.sol (excerpt)" %}

```
```

{% endcode %}

```solidity
    function addPrize(
        string calldata name,
        string calldata description,
        uint256 value,
        uint256 quantity
    ) external onlyRole(ADMIN_ROLE) {
        uint256 prizeId = nextPrizeId++;
        prizeIds.push(prizeId);

        require(bytes(prizes[prizeId].name).length == 0, "Prize ID already in use");
        require(quantity > 0, "Quantity must be greater than 0");

        prizes[prizeId] = Prize({ 
            name: name,
            description: description,
            value: value,
            endTimestamp: 0,
            isActive: true,
            winner: address(0), // deprecated
            winnerIndex: 0, // deprecated
            claimed: false, // deprecated
            quantity: quantity
        });

        emit PrizeAdded(prizeId, name);
    }
```

This function should implement a check that ensures admin doesn't mistakenly set the prize to some large value that the contract cannot afford to offer.

### Impact Details

Mistakenly or intentionally setting the prize to a value that the contract cannot afford to offer would result in the contract failing to deliver the promised payout: when a user tries to claim the prize, the transaction would revert due to insufficient funds. This effectively prevents users from receiving their winnings.

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L128>

## Proof of Concept

{% stepper %}
{% step %}

### Step

The admin sets a prize to a value that the contract cannot afford to offer (example: $10m). This could be malicious or accidental.
{% endstep %}

{% step %}

### Step

A user places a bet and wins the (overstated) $10m prize.
{% endstep %}

{% step %}

### Step

The user attempts to claim the prize, but the claim transaction reverts due to insufficient contract funds.
{% endstep %}

{% step %}

### Step

The user is unable to receive the prize—effectively scammed by the contract's inability to pay the declared prize.
{% endstep %}
{% endstepper %}
