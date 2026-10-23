# 53015 sc low raffle does not invalidate used tickets breaking fairness

* Report ID: #53015
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* Impacts: Business Logic Flaw breaking core functionality

## Description

### Brief / Intro

The raffle system allows the same ticket number to win multiple times across different winner selections. This violates the intended fairness principle where each ticket should only have one chance to win per prize.

### Vulnerability Details

In the `handleWinnerSelection` function, the winner is determined by selecting a `winningTicketIndex` using:

```solidity
uint256 winningTicketIndex = (rng[0] % totalTickets[prizeId]) + 1;
```

A binary search is then performed over `prizeRanges` to find the corresponding user:

```solidity
uint256 lo = 0;
uint256 hi = ranges.length - 1;
while (lo < hi) {
    uint256 mid = (lo + hi) >> 1;
    if (winningTicketIndex <= ranges[mid].cumulativeEnd) {
        hi = mid;
    } else {
        lo = mid + 1;
    }
}
winnerAddress = ranges[lo].user;
```

However, after a ticket wins, it is not removed or marked as ineligible from the ranges. This means that if the VRF oracle later returns a random number that maps to a previously won ticket, the same user can win again for the same prize using the exact same ticket.

The likelihood of generating the same `winningTicketIndex` increases when there are fewer participants and a smaller total number of tickets, because `winningTicketIndex = (randomNumber % totalTickets) + 1` produces values in a limited range.

Example:

* Alice owns tickets 1–5, Bob owns tickets 6–10.
* First draw: `randomNumber = 129483392 → winningTicketIndex = (129483392 % 10) + 1 = 3` → Alice wins.
* Second draw: `randomNumber = 742059642 → winningTicketIndex = (742059642 % 10) + 1 = 3` → Alice wins again, even though the same ticket was already used.

This breaks the expected raffle logic where each ticket should be eligible for at most one win per prize.

## Impact Details

Early-round winners gain an unfair advantage by getting a chance to win again with previously used tickets, reducing the probability of other participants winning in subsequent draws. The current logic undermines the core expectation of fairness in the raffle.

## References

handleWinnerSelection - <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol?utm\\_source=immunefi#L238-L284>

## Proof of Concept

<details>

<summary>Proof of Concept</summary>

None

</details>
