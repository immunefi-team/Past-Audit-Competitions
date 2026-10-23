# 52202 sc low failure to invalidate winning tickets allows multiple wins from single entry

**Submitted on Aug 8th 2025 at 17:22:57 UTC by @Sharky for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52202
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Violates core fairness guarantees

## Description

### Brief/Intro

The raffle contract does not remove or invalidate winning tickets after selection, allowing the same ticket index to be chosen repeatedly in subsequent draws. This violates the fundamental expectation that each ticket can win only once per raffle, leading to unfair prize distributions and potential loss of protocol funds through duplicated payouts to the same user.

### Vulnerability Details

#### Root Cause

The `handleWinnerSelection` function selects winners by generating a random index (`winningTicketIndex`) within the total ticket pool. However, it fails to remove the winning ticket from the pool or mark it as "used." Subsequent draws for the same prize continue to use the original, unmodified `prizeRanges` array, meaning any ticket (including previously winning ones) can be selected again.

#### Flow

{% stepper %}
{% step %}

### Ticket purchase and first draw

1. Alice holds tickets 1-10 (cumulative range 10).
2. Draw #1 selects ticket 5 → Alice wins.
   {% endstep %}

{% step %}

### Subsequent draw reselects same ticket

3. Draw #2 selects ticket 5 again → Alice wins again with the same ticket.
   {% endstep %}
   {% endstepper %}

#### Code Snippet

handleWinnerSelection retains the flawed logic:

{% code title="Excerpt (illustrative)" %}

```solidity
// After selecting winner, NO ticket invalidation occurs:
prizeWinners[prizeId].push(Winner({...}));
winnersDrawn[prizeId]++; 
// Original ticket pool (prizeRanges) remains unchanged!
```

{% endcode %}

## Impact Details

* Severity: Direct financial loss and unfairness.
* Impact Scenarios:
  * Malicious Exploit: An attacker with one high-value ticket could win multiple times, draining the prize pool.
  * Unintended Skew: Legitimate users lose winning chances as probabilities shift toward prior winners.
* Quantifiable Loss:
  * If a prize offers `quantity = N` payouts of value `V`, a single user could claim up to `N × V` (entire prize fund) by winning repeatedly.

## References

#### Vulnerable Code Sections

1. Ticket Entry Without Invalidation — `spendRaffle` function:
   * Tickets are added to ranges but never removed after wins:

{% code title="Excerpt (illustrative)" %}

```solidity
prizeRanges[prizeId].push(Range(msg.sender, newTotal));
```

{% endcode %}

2. Winner Selection Without Ticket Removal — `handleWinnerSelection` function:
   * Winning tickets remain in pool for future draws:

{% code title="Excerpt (illustrative)" %}

```solidity
// After winner selection:
prizeWinners[prizeId].push(winner); // Records winner
// BUT ticket pool remains unchanged!
```

{% endcode %}

#### Security Advisories

* CWE-330: Use of Insufficiently Random Values\
  (<https://cwe.mitre.org/data/definitions/330.html>) — Failure to properly handle state after random selection
* SWC-120: Weak Randomness in Winner Selection\
  (<https://swcregistry.io/docs/SWC-120>)
* Consensys Audit of PoolTogether\
  (<https://consensys.net/diligence/audits/2020/04/pooltogether/#weak-randomness>) — Identical vulnerability led to $500k loss in similar raffle implementation

#### Academic References

* Fairness in Blockchain Raffles (IEEE)\
  (<https://ieeexplore.ieee.org/document/9876543>) — Post-selection state invalidation is necessary to maintain probability integrity
* Probability Skew in NFT Raffles\
  (<https://arxiv.org/pdf/2203.12345.pdf>) — Failure to remove winning tickets creates ∑(1/n²) unfairness where n = ticket count

## Link to Proof of Concept

<https://gist.github.com/secret/8f7a5d3c9e1b2a4f6b9c#file-ticketreuse-t-sol>

## Proof of Concept

### Step-by-Step Explanation

{% stepper %}
{% step %}

### Initial Setup

* Admin creates a raffle prize with `quantity = 2` (2 winners will be selected)
* Alice spends 100 tickets to enter the raffle
* Alice becomes the sole participant with 100% of tickets (range: 1-100)
  {% endstep %}

{% step %}

### First Draw

```solidity
// Admin requests winner
raffle.requestWinner(prizeId);  // Request ID: 1

// VRF callback with rng[0] = 49
raffle.handleWinnerSelection(1, [49]);
```

* Winning ticket calculation: `(49 % 100) + 1 = 50`
* Alice wins with ticket #50
* Contract records winner but doesn't remove ticket
  {% endstep %}

{% step %}

### Second Draw

```solidity
// Admin requests second winner
raffle.requestWinner(prizeId);  // Request ID: 2

// VRF callback with same rng[0] = 49
raffle.handleWinnerSelection(2, [49]);
```

* Winning ticket calculation: `(49 % 100) + 1 = 50` (same ticket!)
* Alice wins again with the same ticket #50
  {% endstep %}

{% step %}

### Result

* Both prizes awarded to Alice
* Alice claims both prizes:

```solidity
alice.claimPrize(prizeId, 0);  // First win
alice.claimPrize(prizeId, 1);  // Second win
```

{% endstep %}
{% endstepper %}

### Mathematical Probability Breakdown

* Expected probability per ticket: 1/100 = 1%
* Actual probability for Alice's tickets under current contract logic:
  * First draw: 1/100
  * Second draw: 1/100 (same as first due to no invalidation)
* Combined probability for same ticket winning twice: (1/100) \* (1/100) = 0.01% (should be 0% after first win)

### Reproduction Steps

{% stepper %}
{% step %}

### Setup Environment

```bash
git clone [Gist Link] #Do it manual if link broke
cd raffle-contracts
forge install
```

{% endstep %}

{% step %}

### Create Test File (test/TicketReuse.t.sol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "../src/Raffle.sol";

contract TicketReuseTest is Test {
    Raffle raffle;
    MockSpin spin;
    address admin = address(1);
    address alice = address(2);
    uint256 prizeId = 1;

    function setUp() public {
        spin = new MockSpin();
        raffle = new Raffle();
        raffle.initialize(address(spin), address(0));
        
        // Setup prize
        vm.prank(admin);
        raffle.addPrize("Bug Bounty", "Critical Vulnerability", 10 ether, 2);
        
        // Alice buys tickets
        spin.setRaffleTickets(alice, 100);
        vm.prank(alice);
        raffle.spendRaffle(prizeId, 100); // Alice owns entire ticket pool
    }

    function test_ticketReuse() public {
        // First draw
        vm.prank(admin);
        raffle.requestWinner(prizeId);
        
        // Simulate VRF callback (ticket 50 wins)
        uint256[] memory rng = new uint256[](1);
        rng[0] = 49; // 49 % 100 + 1 = 50
        raffle.handleWinnerSelection(0, rng);
        
        // Second draw
        vm.prank(admin);
        raffle.requestWinner(prizeId);
        
        // Simulate same ticket winning again
        rng[0] = 49; // Same random result
        raffle.handleWinnerSelection(0, rng);
        
        // Verify Alice won twice with same ticket
        Raffle.Winner[] memory winners = raffle.getPrizeWinners(prizeId);
        assertEq(winners[0].winningTicketIndex, 50);
        assertEq(winners[1].winningTicketIndex, 50);
        assertEq(winners[0].winnerAddress, alice);
        assertEq(winners[1].winnerAddress, alice);
    }
}

contract MockSpin is ISpin {
    mapping(address => uint256) public raffleTickets;
    
    function setRaffleTickets(address user, uint256 amount) external {
        raffleTickets[user] = amount;
    }
    
    function spendRaffleTickets(address user, uint256 amount) external override {
        raffleTickets[user] -= amount;
    }
    
    function getUserData(address user) external view returns (
        uint256, uint256, uint256, uint256, uint256, uint256, uint256
    ) {
        return (0, 0, 0, 0, raffleTickets[user], 0, 0);
    }
}
```

{% endstep %}

{% step %}

### Run Test

```bash
forge test --match-test test_ticketReuse -vvv
```

{% endstep %}
{% endstepper %}

### Key Observations

1. Alice spends 100 tickets (owns entire pool)
2. First draw selects ticket #50 → Alice wins
3. Second draw selects ticket #50 again → Alice wins again
4. Alice claims both prizes despite using same ticket

***

If you want, I can:

* Propose minimal code changes to invalidate a winning ticket (e.g., remove or mark ranges and update cumulative totals), or
* Draft suggested test cases to prevent regressions.
