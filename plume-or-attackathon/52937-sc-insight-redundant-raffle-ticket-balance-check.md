# 52937 sc insight redundant raffle ticket balance check

**Submitted on Aug 14th 2025 at 12:15:38 UTC by @Am3nh3l for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52937
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**

## Description

### Brief/Intro

The Spin contract contains a redundant balance check in its `spendRaffleTickets` function that duplicates functionality already implemented in the Raffle contract. This creates unnecessary gas overhead and code redundancy without adding security benefits.

### Vulnerability Details

The Raffle contract already performs a sufficient ticket balance check before calling `spendRaffleTickets`:

```solidity
// Raffle.sol - Line 194
(,,,, uint256 userRaffleTickets,,) = spinContract.getUserData(msg.sender);
if (userRaffleTickets < ticketAmount) revert InsufficientTickets();
```

Despite this, the Spin contract repeats the same check:

```solidity
function spendRaffleTickets(address user, uint256 amount) external onlyRaffleContract {
    UserData storage userDataStorage = userData[user];
    require(userDataStorage.raffleTicketsBalance >= amount, "Insufficient raffle tickets");
    // ...
}
```

This redundancy exists because:

* The Raffle contract is the only caller (via `onlyRaffleContract`)
* The Raffle contract already verifies sufficient balance
* The Spin contract cannot be called independently for ticket spending

## Impact Details

(Impact details not provided in original report.)

## References

(Add any relevant links to documentation or code)

## Proof of Concept

(Proof of Concept not provided in original report.)

## Proof of Concept

Insight

<details>

<summary>Raw target file</summary>

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>

</details>
