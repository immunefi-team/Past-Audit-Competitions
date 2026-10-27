# 51863 sc low lack of winning ticket removal in handlewinnerselection leads to unfair prize distribution and economic exploitation

**Submitted on Aug 6th 2025 at 10:13:56 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51863
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `handleWinnerSelection` function fails to remove or mark winning tickets after selection, allowing the same ticket number to win multiple times in multi-winner raffles.

This flaw allows a user with a single ticket entry to potentially win all available prizes for a given raffle, breaking the fairness and economic model of the raffle system where prizes should be distributed among different participants.

### Vulnerability Details

The vulnerability exists in the winner selection mechanism which doesn't prevent previously-won tickets from winning again:

```solidity
function handleWinnerSelection(uint256 requestId, uint256[] memory rng) external onlyRole(SUPRA_ROLE) {
    uint256 prizeId = pendingVRFRequests[requestId];
    
    // ... validation ...
    
    // Calculate winning ticket - same range every time
    uint256 winningTicketIndex = (rng[0] % totalTickets[prizeId]) + 1;
    
    // Binary search to find winner
    Range[] storage ranges = prizeRanges[prizeId];
    address winnerAddress;
    
    // ... binary search logic ...
    
    // Store winner - but ticket remains in pool
    prizeWinners[prizeId].push(Winner({
        winnerAddress: winnerAddress,
        winningTicketIndex: winningTicketIndex,
        drawnAt: block.timestamp,
        claimed: false
    }));
    
    // Increment counters but don't modify ticket pool
    winnersDrawn[prizeId]++;
    userWinCount[prizeId][winnerAddress]++;
}
```

The critical issues are:

* No Ticket Removal: `totalTickets[prizeId]` remains unchanged after each draw
* No Ticket Marking: No mechanism tracks which tickets have already won
* Static Pool: `prizeRanges[prizeId]` array remains unmodified between draws
* Same Probability Space: Each draw uses identical ticket range (1 to `totalTickets`)

This design flaw means the same ticket index can be selected repeatedly across multiple winner draws.

## Impact Details

A user with just one ticket entry could win all prizes:

Example:

* 10 Nintendo Switch raffle (quantity = 10)
* User buys 1 ticket (#500)
* If RNG repeatedly selects 500, the user wins all 10 units

Breakdown of Trust:

* Users expect fair distribution among participants
* If one ticket wins multiple times, credibility is destroyed
* This is unfair prize distribution and economic exploitation of the raffle system

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L237-L284>

## Proof of Concept

{% stepper %}
{% step %}

### Initial Setup

```
Prize ID: 1 (PlayStation 5)
Quantity: 3 units
Participants:
- Alice: Tickets 1-100 (100 tickets)
- Bob: Tickets 101-600 (500 tickets)  
- Carol: Tickets 601-1000 (400 tickets)
Total tickets: 1000
```

{% endstep %}

{% step %}

### First Winner Draw

```
1. Admin calls requestWinner(1)
2. VRF returns random number: 7,234,567
3. handleWinnerSelection calculates:
   winningTicket = 7234567 % 1000 + 1 = 568
4. Binary search finds: Bob owns ticket 568
5. Result: Bob wins 1st PS5
6. State: winnersDrawn[1] = 1

Important: totalTickets[1] still = 1000
          Ticket 568 still owned by Bob
```

{% endstep %}

{% step %}

### Second Winner Draw

```
1. Admin calls requestWinner(1)
2. VRF returns random number: 9,345,567
3. handleWinnerSelection calculates:
   winningTicket = 9345567 % 1000 + 1 = 568
4. Binary search finds: Bob owns ticket 568 (SAME TICKET!)
5. Result: Bob wins 2nd PS5
6. State: winnersDrawn[1] = 2
```

{% endstep %}

{% step %}

### Third Winner Draw

```
1. Admin calls requestWinner(1)
2. VRF returns random number: 4,789,567
3. handleWinnerSelection calculates:
   winningTicket = 4789567 % 1000 + 1 = 568
4. Binary search finds: Bob owns ticket 568 (SAME TICKET AGAIN!)
5. Result: Bob wins 3rd PS5
6. State: winnersDrawn[1] = 3, prize becomes inactive
```

{% endstep %}
{% endstepper %}

Final Result:

```solidity
prizeWinners[1] = [
    Winner(Bob, 568, timestamp1, false),
    Winner(Bob, 568, timestamp2, false),
    Winner(Bob, 568, timestamp3, false)
]

Bob's single ticket #568 won all 3 PlayStation 5s!
```

Verification of the Issue:

* `totalTickets[prizeId]` never decreases after wins
* `prizeRanges[prizeId]` is never modified after wins
* No mapping exists to track used tickets
* The same `winningTicketIndex` can appear multiple times in `prizeWinners`
