# #45336 \[SC-Low] Malicious Agent could repeatedly create and destroy vaults reserving different suffixes and grief other agents

**Submitted on May 12th 2025 at 20:00:55 UTC by @avoloder for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45336
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-labs-ltd/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A malicious agent, once whitelisted could create multiple vaults just to reserve different suffixes, so that other users cannot use them

## Vulnerability Details

During the creation of agent vaults, user provides agent settings, among which is also `poolTokenSuffix`. During the vault creation (as part of the `createAgentVault()` function), this `poolTokenSuffix` gets validated and reserved by saving it in the mapping `reservedPoolTokenSuffixes`. Before being saved, it is checked if the suffix is already present in the mapping and only saved if it's not.

User can also destroy the vault at any time. However, the mapping `reservedPoolTokenSuffixes` is never updated, i.e., the suffix remains active. This way, a malicious user, could repeatedly create vaults, reserve different suffixes (for different tokens, different combinations) and therefore prevent other legitimate agents to choose suffix they want. User could then destroy the vault and get back all his funds, so the only costs of the attack are gas costs.

## Impact Details

The impact is griefing -> causing harm to legit users (agents) by reserving different suffixes at almost no cost.

## References

<https://github.com/flare-labs-ltd/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsCreateDestroy.sol#L59-L121> <https://github.com/flare-labs-ltd/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsCreateDestroy.sol#L239-L256> <https://github.com/flare-labs-ltd/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentsCreateDestroy.sol#L145-L176>

## Proof of Concept

## Proof of Concept

Run this test as part of the `Agents.ts`

```
it("can reserve multiple suffixes by creating and destroying vault", async () => {
        const suffix = "SUFFX1";

        assert.isFalse(await assetManager.isPoolTokenSuffixReserved(suffix));
        const agentVault = await createAgent(agentOwner1, underlyingAgent1, { poolTokenSuffix: suffix });

        // announce destroy agent
        await assetManager.announceDestroyAgent(agentVault.address, { from: agentOwner1 });
        await deterministicTimeIncrease(300);

        const tx = await assetManager.destroyAgent(agentVault.address, agentOwner1, { from: agentOwner1 });
        assert.isTrue(await assetManager.isPoolTokenSuffixReserved(suffix));
        await expectRevert(createAgent(agentOwner1, "Agent2", { poolTokenSuffix: suffix }), "suffix already reserved");
    })
```
