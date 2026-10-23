# #46886 \[SC-Low] \`destroyAgent()\` functionality can easily be bricked due to Frontrunning Attack

**Submitted on Jun 5th 2025 at 21:15:46 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46886
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Description

The agent destruction process is designed to safely decommission agents by enforcing a waiting period after which the agent can now call the `destroyAgent()` function to finalize the whole process. Now this function makes an external call to the `CollateralPool` to destroy it:

```solidity
 function destroy(address payable _recipient)
        external
        onlyAssetManager
        nonReentrant
    {
@>>        require(token.totalSupply() == 0, "cannot destroy a pool with issued tokens");
        token.destroy(_recipient);
        // transfer native balance, if any (used to be done by selfdestruct)
        Transfers.transferNAT(_recipient, address(this).balance);
        // transfer untracked f-assets and wNat, if any
     .............................
    }
```

There exist a condition that ensures no owned funds remain in the collateral pool (totalSupply == 0). This invariant is critical for maintaining the integrity and finality of the agent lifecycle.

However, a vulnerability exists in the `CollateralPool` contract due to the lack of a state check in the `enter()` function. Specifically, `enter()` does not verify whether the associated agent is currently in the `DESTROYING` state before allowing deposits.

This opens a griefing vector: an attacker can frontrun the `destroyAgent()` transaction by depositing a small amount of collateral via `enter()`. This breaks the destruction condition (since `totalSupply > 0`), causing the agent’s `destroy()` call to fail. After the failure, the attacker can simply withdraw their funds using `exit()`.

Since this attack does not require significant capital and can be repeated indefinitely, it effectively allows malicious actors to prevent agents from ever exiting the system.

## Impact Details

This attack makes it impossible for agents to safely exit the system, since the Agents are permanently or repeatedly blocked from exiting the system.

## Recommendations

Add a check in the `enter()` function to ensure that no new deposits can be made if the agent is in the `DESTROYING` state:

```solidity
function enter(uint256 _fAssets, bool _enterWithFullFAssets)
    external payable
    nonReentrant
    returns (uint256, uint256)
{
    Agent.State storage agent = Agent.get(agentVault);
    require(agent.status != Agent.Status.DESTROYING, "agent is being destroyed");
    
    AssetData memory assetData = _getAssetData();
    // ... rest of the logic
}
```

A more efficient recommendation is to not bother about the remaining total supply when destorying agent vault since it's a timelock operation and agents have made announcement already legit users should have removed their funds safely before then, most likely the remaining share supply are dead shares.

## Proof of Concept

## Proof of Concept

This is the attack path:

1. Agent announces destruction via `announceDestroyAgent`; status changes to DESTROYING and a waiting period begins.
2. After some time the agent calls the `destroyAgent()` to finalize the process
3. Attacker frontruns this process by calling `enter()` with a minimal deposit into the `collateralPOOL`.
4. When the agent later attempts to finalize destruction via `destroy()`, the call fails due to totalSupply > 0.
5. Attacker backruns and calls `exit()` to withdraw their deposit.
