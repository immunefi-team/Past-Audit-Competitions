# #46714 \[SC-Medium] Agent can frontrun executor to steal unclaimed executor fee in minting process

**Submitted on Jun 3rd 2025 at 20:55:07 UTC by @avoloder for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46714
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

Agents can frontrun executors in the minting process to steal their fees by calling `executeMinting` first, causing the executor's fee to be distributed as part of the collateral reservation fee instead of being paid to the executor.

## Vulnerability Details

When a user reserves collateral for minting (`CollateralReservations.sol`), any excess funds sent beyond the required reservation fee are designated as the executor's fee.

However, the executor fee is only paid if the executor themselves calls `executeMinting` (`Minting.sol`). If the `msg.sender` is not the executor, the fee will be distributed to the agent's collateral pool and to the agent's vault address.

The issue lies within the fact that `executeMinting` can also be called by the agents themselves, as a safety mechanism to release their collateral if the minter becomes unresponsive. This gives agents the option to frontrun the executor's transaction and steal their unclaimed fee since there is no time delay or waiting period that would protect the executor.

## Impact Details

The impact is high as the unclaimed fees from the executors could easily be stolen by the agent

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol#L57-L71>

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/Minting.sol#L61-L68>

## Proof of Concept

## Proof of Concept

1. User A reserves collateral with excess NAT, designating User B as executor
2. User A makes the required payment
3. User B attempts to execute minting to claim their fee
4. Agent frontruns User B's transaction
5. Agent's transaction executes first, distributing executor fee to agent's pools
6. User B's transaction fails and User B does not receive his/her fees
