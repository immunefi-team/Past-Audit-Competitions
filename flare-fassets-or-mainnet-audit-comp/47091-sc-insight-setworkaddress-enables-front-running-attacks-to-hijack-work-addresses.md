# #47091 \[SC-Insight] \`setWorkAddress()\` enables front-running attacks to hijack work addresses

**Submitted on Jun 8th 2025 at 21:28:27 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47091
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentOwnerRegistry.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Description

The AgentOwnerRegistry contract implements a `setWorkAddress()` function that allows an agent to designate a secondary "work" address associated with their management address. However, the current implementation does not sufficiently validate ownership or authority over the work address, creating a front-running vector that allows malicious agents with no potential risk (active vault) to hijack work addresses before legitimate owners can claim them.

This introduces a critical denial-of-service condition and opens up potential for targeted griefing attacks against agents attempting to onboard or reconfigure their setup.

```solidity
function setWorkAddress(address _ownerWorkAddress) external {
    require(isWhitelisted(msg.sender),
        "agent not whitelisted");
    require(_ownerWorkAddress == address(0) || workToMgmtAddress[_ownerWorkAddress] == address(0),
        "work address in use");
    // ...
}
```

The function checks that the work address is currently unused, creating a room for hijacking the work address of legitimate agent and preventing them for setting their configuration.

## Impact Details

Malicious agent `Alice` can keep frontrunning whitelisted agent `Bob` when she notice `Bob` wants to set work address to specific value. This leads to denial-of-service attack on `Bob` indefinitely.

## Recommendations

I think it's better to introduce an off-chain mechanism to oversee the setting of the `workAddress` rather than allowing a specific agent to set up the `workAddress` from their end, this gives a constraint to random and frequent change of the `workAddress`.

## Proof of Concept

## Proof of Concept

Consider this straightforward attack scenario:

1. A legitimate agent attempts to register their work address:

```
agent.setWorkAddress(0xLegitimateWorkAddress);
```

2. A malicious actor monitors the mempool and sees this transaction.
3. The malicious actor quickly front-runs the transaction:

```
attacker.setWorkAddress(0xLegitimateWorkAddress);
```

4. The original agent’s transaction reverts with "work address in use".

This allows the attacker to indefinitely block the legitimate agent from using their desired work address.
