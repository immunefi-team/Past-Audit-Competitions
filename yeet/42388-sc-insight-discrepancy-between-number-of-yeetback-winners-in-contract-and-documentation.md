# #42388 \[SC-Insight] Discrepancy between number of Yeetback winners in contract and documentation

**Submitted on Mar 23rd 2025 at 14:12:20 UTC by @Oxrochimaru for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42388
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeetback.sol>
* **Impacts:**

## Description

## Brief/Intro

As per documentation the number of yeetback winners are 10 but are configurable via config. But as per `Yeetback::draftWinners()` **nrOfWinners** are hardcoded to 10 and can't be changed.

<https://docs.yeetit.xyz/yeet/yeet-game>

```solidity
    function draftWinners(uint256 randomNumber, uint256 round) private {
        uint256 potValue = potForRound[round];
        uint256 nrOfYeets = yeetsInRound[round].length;
        uint256 nrOfWinners = 10;

        uint256 winnings = potValue / nrOfWinners;
        amountToWinners[round] = winnings;

        for (uint256 i; i < nrOfWinners; i++) {
            uint256 randomDataNumber = uint256(keccak256(abi.encodePacked(randomNumber, i)));
            uint256 winningYeetIndex = randomDataNumber % nrOfYeets; // index of the winning yeet
            address winnerAddress = yeetsInRound[round][winningYeetIndex];

            // Update amountToWinners and amountOfWins
            amountOfWins[round][winnerAddress] += 1;

            emit YeetbackWinner(round, winnerAddress, winnings, winningYeetIndex);
        }
    }
```

## Vulnerability Details

The documentation states that number of winners can change but it is hardcoded to 10 in the smart contract.

## Impact Details

Documentation is contradicting the actual behaviour in the code.

## References

`Numbers that are formated in this way 10 means that they are changeable via some configuration.`

## Proof of Concept

## Proof of Concept

* User reading the documentation will believe number of winners are configurable.
* Actually no. of winners are fixed to 10 in the smart contract.
