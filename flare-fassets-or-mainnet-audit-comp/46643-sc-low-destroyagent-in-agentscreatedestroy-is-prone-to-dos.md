# #46643 \[SC-Low] \`destroyAgent\` in \`AgentsCreateDestroy\` is prone to DOS

**Submitted on Jun 2nd 2025 at 19:15:32 UTC by @ni8mare for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46643
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/AgentsCreateDestroy.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`destroyAgent` in `AgentsCreateDestroy` is prone to DOS. An attacker can prevent an agent vault owner from destroying their collateral pool and vault pool, thereby preventing them from sending funds from the pools to the desired recipient.

## Vulnerability Details

An agent calls `announceDestroy` with the intention of destroying their vaults and transferring all the funds from them.

After enough time has passed, the agent then calls the `destroyAgent` function:

```
    function destroyAgent(
        address _agentVault,
        address payable _recipient
    )
        internal
        onlyAgentVaultOwner(_agentVault)
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Agent.State storage agent = Agent.get(_agentVault);
        // destroy must have been announced enough time before
        require(agent.status == Agent.Status.DESTROYING, "destroy not announced");
        require(block.timestamp > agent.destroyAllowedAt, "destroy: not allowed yet");
        // cannot have any minting when in destroying status
        assert(agent.totalBackedAMG() == 0);
        // destroy pool
        agent.collateralPool.destroy(_recipient);
        // destroy agent vault
        IIAgentVault(_agentVault).destroy(_recipient);
        // remove from the list of all agents
        uint256 ind = agent.allAgentsPos;
        if (ind + 1 < state.allAgents.length) {
            state.allAgents[ind] = state.allAgents[state.allAgents.length - 1];
            Agent.State storage movedAgent = Agent.get(state.allAgents[ind]);
            movedAgent.allAgentsPos = uint32(ind);
        }
        state.allAgents.pop();
        // delete agent data
        AgentSettingsUpdater.clearPendingUpdates(agent);
        Agent.deleteStorage(agent);
        // notify
        emit IAssetManagerEvents.AgentDestroyed(_agentVault);
    }
```

This calls the `agent.collateralPool.destroy(_recipient);` function:

```
    function destroy(address payable _recipient)
        external
        onlyAssetManager
        nonReentrant
    {
     -->   require(token.totalSupply() == 0, "cannot destroy a pool with issued tokens");
        token.destroy(_recipient);
        // transfer native balance, if any (used to be done by selfdestruct)
        Transfers.transferNAT(_recipient, address(this).balance);
        // transfer untracked f-assets and wNat, if any
        uint256 untrackedWNat = wNat.balanceOf(address(this));
        uint256 untrackedFAsset = fAsset.balanceOf(address(this));
        if (untrackedWNat > 0) {
            wNat.safeTransfer(_recipient, untrackedWNat);
        }
        if (untrackedFAsset > 0) {
            fAsset.safeTransfer(_recipient, untrackedFAsset);
        }
    }
```

An attacker can easily abuse this check: `require(token.totalSupply() == 0, "cannot destroy a pool with issued tokens");`

All they need to do is call the `enter` function and mint the collateral pool tokens. This would increase the `totalSupply()`, which would make the function revert.

Thus, an attacker in this way can easily DOS the destroy function.

## Impact Details

The `destroy` function can never be called because of this. The agent is never able to destroy their vaults. This also prevents the agents from transferring the funds from the collateral pool and the vault pool to themselves.

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/AgentsCreateDestroy.sol#L145>

## Proof of Concept

## Proof of Concept

1. Agent calls the `announceDestroy` function with the intention of destroying their vaults.
2. Before the agent tries to call `destroyAgent` function, an attacker calls the `enter` function of the collateral pool.
3. He mints the pool tokens by doing so and hence increases the `totalSupply()`.
4. When enough time has passed, the agent tries to call `destroyAgent` function. But it reverts. This is because of the check that requires `totalSupply()` to be 0, which is no longer the case.
