# #46721 \[SC-Insight] Inconsistencies for agentTimelockedOperationWindowSeconds value checks between SettingsInitializer.sol::\_validateSettings and SettingsManagementFacet.sol::setAgentTimelockedOpera...

**Submitted on Jun 3rd 2025 at 23:18:22 UTC by @hunter0xweb3 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46721
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/SettingsInitializer.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

There is an inconsistency between requirements check for minimum value for agentTimelockedOperationWindowSeconds in SettingsInitializer.sol::\_validateSettings and SettingsManagementFacet.sol::setAgentTimelockedOperationWindowSeconds functions leading to protocol invariant conditionals breakage as shown below

## Vulnerability Details

Protocol defines settings.agentTimelockedOperationWindowSeconds variable to set the time window for agent's validity actions.

Ie once an agent action can be executed (after action was anounced and wait period has passed), agent has\
settings.agentTimelockedOperationWindowSeconds time to execute it before action ability to execute expires:\
Take for example and agent trying to exit (after exit announcement):\
It has settings.agentTimelockedOperationWindowSeconds to execute exit action as checked in \[1]:\
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/AvailableAgents.sol#L73>

```js
    function exit(
        address _agentVault
    )
        internal
        onlyAgentVaultOwner(_agentVault)
    {
        //......
        Agent.State storage agent = Agent.get(_agentVault);
        require(agent.availableAgentsPos != 0, "agent not available");
        require(agent.exitAvailableAfterTs != 0, "exit not announced");
        require(block.timestamp >= agent.exitAvailableAfterTs, "exit too soon");
[1]>    require(block.timestamp <= agent.exitAvailableAfterTs + settings.agentTimelockedOperationWindowSeconds,
            "exit too late");
```

Or while withdrawing tokens same variable is checked:\
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/AgentsExternal.sol#L124>

```

    function beforeCollateralWithdrawal(
        IERC20 _token,
        address _agentVault,
        uint256 _amountWei
    )
        internal
    {
        Agent.State storage agent = Agent.get(_agentVault);
        Collateral.Kind kind;
	//...
@>        require(block.timestamp <= withdrawal.allowedAt + settings.agentTimelockedOperationWindowSeconds,
            "withdrawal: too late");
```

The issue occurs because there are inconsistencies between checks when setting settings.agentTimelockedOperationWindowSeconds value.

contracts/assetManager/library/SettingsInitializer.sol::\_validateSettings checks that agents has at least one hour as a time window to execute its announced action:\
This checks are performed on settings initialization, ie global settings set up: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/SettingsInitializer.sol#L41>

```js
    function _validateSettings(
        AssetManagerSettings.Data memory _settings
    )
        private pure
    {
        require(_settings.fAsset != address(0), "zero fAsset address");
        require(_settings.agentVaultFactory != address(0), "zero agentVaultFactory address");
        //...
        //...
@>      require(_settings.agentTimelockedOperationWindowSeconds >= 1 hours, "value too small");
```

However contracts/assetManager/facets/SettingsManagementFacet.sol allows to set settings.agentTimelockedOperationWindowSeconds value as low as 1 minute:\
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/facets/SettingsManagementFacet.sol#L557-L566>

```js
    function setAgentTimelockedOperationWindowSeconds(uint256 _value)
        external
        onlyAssetManagerController
        rateLimited
    {
        AssetManagerSettings.Data storage settings = Globals.getSettings();
        // validate
@>        require(_value >= 1 minutes, "value too small");
        // update
        settings.agentTimelockedOperationWindowSeconds = _value.toUint64();
        emit SettingChanged("agentTimelockedOperationWindowSeconds", _value);
    }
```

Leading to a really reduced time action window for agents making agents unable to perform anounced actions (DoS) and breaking the conditionals imposed on protocol settings initialization (settings.agentTimelockedOperationWindowSeconds value must be >= 1 hour)

## Impact Details

Bypassing restriction and protocol settings initialization for settings.agentTimelockedOperationWindowSeconds (value must be >= 1 hour)\
DoS agents ability to execute action (exiting | withdrawing collateral) bypassing SettingsInitializer.sol::\_validateSettings minimum value with SettingsManagementFacet.sol::setAgentTimelockedOperationWindowSeconds

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/SettingsInitializer.sol#L41\\>
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/facets/SettingsManagementFacet.sol#L557-L566>

## Proof of Concept

## Proof of Concept

Protocol is initiazed with settings.agentTimelockedOperationWindowSeconds = 1 hour (minimum accepted value) giving enough time for agents to execute actions (exit , withdraw, etc)\
Later SettingsManagementFacet.sol::setAgentTimelockedOperationWindowSeconds is used to set settings.agentTimelockedOperationWindowSeconds = 1 minute
