# #47094 \[SC-Insight] Missing Event Emission in \`AgentVault\` and \`CollateralPoolToken\` Factory Contracts

**Submitted on Jun 8th 2025 at 22:27:50 UTC by @blackgrease for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47094
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVaultFactory.sol>
* **Impacts:**

## Description

#### Note: Due to the issue being prevalent in two contracts but maintaining the same fix - and considering this is an Insight report - the two occurrences have been merged into one report.

#### Severity > Insight: Code Enhancements and Optimization and/or Best Practice

## Summary

The two factory contracts, `CollateralPoolTokenFactory` and `AgentVaultFactory`, do not emit events upon successful creation of new contract instances. This omission reduces on-chain transparency and makes it difficult for off-chain systems to track deployed contracts. It further breaks the best practive of events being emitted on important state changes.

## Description

Both `CollateralPoolTokenFactory` and `AgentVaultFactory` are responsible for deploying new proxy contract instances (CollateralPoolToken and AgentVault, respectively) using the ERC1967Proxy pattern.

However, neither factory contract emits an event upon creation of a new contract. This breaks a smart contract development best practice where a contract should emit an event on important state changes; such as a new contract being created, important state variable change and such. This helps to notify off-chain infrastructure (e.g indexers, explorers, front-ends) about new deployments.

> Affected Areas

* CollateralPoolTokenFactory

```solidity

function create(IICollateralPool _pool, string memory _systemSuffix, string memory _agentSuffix)
        external override
        returns (address)
    {
        string memory tokenName = string.concat(TOKEN_NAME_PREFIX, _systemSuffix, "-", _agentSuffix);
        string memory tokenSymbol = string.concat(TOKEN_SYMBOL_PREFIX, _systemSuffix, "-", _agentSuffix);
        ERC1967Proxy proxy = new ERC1967Proxy(implementation, new bytes(0));
        CollateralPoolToken poolToken = CollateralPoolToken(address(proxy));
        poolToken.initialize(address(_pool), tokenName, tokenSymbol);
        return address(poolToken); //@audit-insight: no event emission
    }
```

* AgentVaultFactory

```solidity

   function create(IIAssetManager _assetManager) external returns (IIAgentVault) {
        ERC1967Proxy proxy = new ERC1967Proxy(implementation, new bytes(0));
        AgentVault agentVault = AgentVault(payable(address(proxy)));
        agentVault.initialize(_assetManager);
        return agentVault; //@audit-insight: no event emission
    }
```

## Impact

This issue is an Insight under Code Optimizations and Enhancements and Best Practices. The results of the missing event emission are:

1. Reduced Transparency: Users, auditors, and developers are unable to verify contract deployments easily.
2. Poor Developer and User Experience: Front-ends cannot reliably fetch and display newly created contract instances for users without resorting to inefficient on-chain scans.
3. Security Concerns: The absence of event logs makes it harder to audit and detect malicious or accidental deployments.

## Mitigation

The recommend mitigation is to include event emission each time a new instance of either `AgentVault` or `CollateralPoolToken` factory is created.

> In `CollateralPoolTokenFactory`

```solidity

//Event Declaration
event CollateralPoolTokenCreated(address indexed poolToken, string tokenName, string tokenSymbol);


function create(IICollateralPool _pool, string memory _systemSuffix, string memory _agentSuffix)
    external override
    returns (address)
{
    string memory tokenName = string.concat(TOKEN_NAME_PREFIX, _systemSuffix, "-", _agentSuffix);
    string memory tokenSymbol = string.concat(TOKEN_SYMBOL_PREFIX, _systemSuffix, "-", _agentSuffix);
    ERC1967Proxy proxy = new ERC1967Proxy(implementation, new bytes(0));
    CollateralPoolToken poolToken = CollateralPoolToken(address(proxy));
    poolToken.initialize(address(_pool), tokenName, tokenSymbol);
    emit CollateralPoolTokenCreated(address(poolToken), tokenName, tokenSymbol); //Event implemented
    return address(poolToken);
}

```

> In `AgentVaultFactory`

```solidity

//Event Declaration
event AgentVaultCreated(address indexed vault, address indexed assetManager);

function create(IIAssetManager _assetManager) external returns (IIAgentVault) {
    ERC1967Proxy proxy = new ERC1967Proxy(implementation, new bytes(0));
    AgentVault agentVault = AgentVault(payable(address(proxy)));
    agentVault.initialize(_assetManager);
    emit AgentVaultCreated(address(agentVault), address(_assetManager)); //Event implemented
    return agentVault;
}

```

## Proof of Concept

## Proof-of-concept

> `CollateralPoolTokenFactory`

1. The function `create` is called on deployed instance of `CollateralPoolFactory`
2. The contract `CollateralPoolToken` is created.
3. The caller gets returned a address value but no event is emitted.
4. No external indexers, observers or off-chain systems are aware that a new `CollateralPoolToken` has been created.
5. Additional step: Caller has to manually make this information public for the system to be aware.

> `AgentVaultFactory`

1. The function `create` is called on deployed instance of `AgentVaultFactory`
2. The contract `AgentVault` is created
3. The caller gets returned a address value but no event is emitted
4. No external indexers, observers or off-chain systems are aware that a new `AgentVault` has been created
5. Additional step: Caller has to manually make this information public aware.
