# 41672 sc insight permanent loss risk of user funds due to inflexible function design in claim&#x20;

## #41672 \[SC-Insight] Permanent Loss Risk of User Funds Due to Inflexible Function Design in Claim()

**Submitted on Mar 17th 2025 at 13:56:22 UTC by @perseverance for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41672
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**
  * Permanent freezing of funds

### Description

## Description

### Brief/Intro

After a game round completed, the winner of a game round in Yeet can claim the reward by calling `claim()` function.

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol#L335-L345>

```solidity
    function claim() external nonReentrant {
        if (winnings[msg.sender] == 0) {
            revert NoWinningsToClaim(msg.sender);
        }

        uint256 valueWon = winnings[msg.sender];
        winnings[msg.sender] = 0;
        (bool success,) = payable(msg.sender).call{value: valueWon}("");
        require(success, "Transfer failed.");
        emit Claim(msg.sender, block.timestamp, valueWon);
    }
```

Or in Yeetback.sol (<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeetback.sol#L121-L134>) there is similar claim function to claim the lottery reward.

### The vulnerability

#### Vulnerability Details

The current implementation of the `claim()` function works for most situations. But there is some edge case when the msg.sender cannot receive native BERA . For example, if the winner of the game or lottery draw is a smart contract without `receive()` or `fallback() payable` functions, then the call `claim()` will always reverted.

**Although this scenario is just an edge case, that is not likely to happen. But if this happened, then nothing can be done for the user to claim the winning BERA.**

## Impacts

### About the severity assessment

This vulnerability is classified as Critical severity because:

Impact category: Permanent freezing of funds

1. It can lead to permanent loss of user funds
2. There is no recovery mechanism
3. The impact is direct financial loss

Although this scenario is an edge case, not likely to happen in most situations. But if it happens, then there is no recovery mechanism. Since this is related to many users in the system, so the risk still exists. It is affecting the main functionality of the Yeet game and the risk of affected fund can be big. For example, right now the pot to winner can be 10\_000 USD.

So to prevent this bug:

1. So you can improve the documentation to highlight this for users. **But this does not eliminate the risk**. So unfortunately if it happens, there is no way to help as the fund is stuck forever.
2. But also you can design the claim() function to accept an address of receiver and allow the winner to claim the BERA sent to that receiver.

**The second way is much better as it allows some flexibility for the winner to claim the reward and eliminated the risk of stuck fund forever.**

Now when we design the system, I recommend to follow the second way.

```solidity
function claim(address payable receiver) external nonReentrant {
    if (winnings[msg.sender] == 0) {
            revert NoWinningsToClaim(msg.sender);
        }

        uint256 valueWon = winnings[msg.sender];
        winnings[msg.sender] = 0;
        (bool success,) = payable(receiver).call{value: valueWon}("");
        require(success, "Transfer failed.");
        emit Claim(msg.sender, block.timestamp, valueWon);
}
```

### Proof of Concept

## Proof of Concept

Step 1: Let's consider a scenario where a smart contract participates in the Yeet game:

```solidity
contract GameParticipant {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function participateInYeet(address yeetContract) external payable {
        // Participate in the game
        IYeet(yeetContract).yeet{value: msg.value}();
    }
    
    // Notice: No receive() or fallback() function
}
```

Step 2: This contract wins the game, and `winnings[contractAddress]` is set to a positive value.

Step 3: When trying to claim:

```solidity
// In GameParticipant contract
function claimWinnings(address yeetContract) external {
    IYeet(yeetContract).claim();  // This will fail as contract cannot receive ETH
}

```

The claim() fails because:

* The contract has no `receive()` or `fallback() payable` function so the BERA (ETH) transfer fails
