# #47087 \[SC-Insight] CollateralTypesFacet.sol::deprecateCollateralType allows to break CollateralTypes.sol::initialize invariant because it allows to deprecate all token collateral vaults leading to ...

**Submitted on Jun 8th 2025 at 20:35:50 UTC by @hunter0xweb3 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47087
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CollateralTypesFacet.sol>
* **Impacts:**

## Description

## Brief/Intro

CollateralTypesFacet.sol::deprecateCollateralType allows to break CollateralTypes.sol::initialize invariant because it allows to remove all token collateral vaults leading to all agents to be liquidated

## Vulnerability Details

When protocol is initialized it calls contracts/assetManager/library/CollateralTypes.sol::initialize with data parameter.\
\_data parameter should contain at least two tokens \[0]:\
The first one should be a pool collateral token \[1] and the rest \[2] (at least one) vault collateral pools:\
contracts/assetManager/library/CollateralTypes.sol::initialize: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralTypes.sol#L14-L19>

```solidity
    function initialize(
        CollateralType.Data[] memory _data
    )
        internal
    {
@>[0]        require(_data.length >= 2, "at least two collaterals required");
        // initial pool collateral token
@>[1]        require(_data[0].collateralClass == CollateralType.Class.POOL, "not a pool collateral at 0");
        _add(_data[0]);
        // ...
        for (uint256 i = 1; i < _data.length; i++) {
@>[2]       require(_data[i].collateralClass == CollateralType.Class.VAULT, "not a vault collateral");
```

So the protocol declares valid pool token and at least one collateral vault token for agent vault\
This collateral vault token function is for backing fassets minted\
If current collateral vault token is deprecated and vault doesnt switch vault collateral before token.isValid , then agent vault can be liquidated

CollateralTypes.sol::initialize ensures at least one vault collateral is enabled for agent vaults

However CollateralTypesFacet.sol::deprecateCollateralType allows to break this invariant allowing to deprecate all vault collateral tokens leaving agent vaults unable to SwitchVaultCollateral and becoming irremediably liquidatable:\
contracts/assetManager/library/CollateralTypes.sol::deprecate:\
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralTypes.sol#L63-L75>

```solidity
    function deprecate(
        CollateralType.Class _collateralClass,
        IERC20 _token,
        uint256 _invalidationTimeSec
    )
        internal
    {
        AssetManagerSettings.Data storage settings = Globals.getSettings();
        CollateralTypeInt.Data storage token = CollateralTypes.get(_collateralClass, _token);
        //...
        uint256 validUntil = block.timestamp + _invalidationTimeSec;
@>      token.validUntil = validUntil.toUint64();
        emit IAssetManagerEvents.CollateralTypeDeprecated(uint8(_collateralClass), address(_token), validUntil);
    }
```

This is due to CollateralTypes.sol::deprecate doesnt check that at least another one valid vault collateral token is available before deprecating the token

### Suggestion

Keep a global variable in state with current number of available vault collaterals tokens\
Increment this variable each time a new token is added with contracts/assetManager/library/CollateralTypes.sol::add\
Decrement this variable each time a token is deprecated with contracts/assetManager/library/CollateralTypes.sol::deprecate first checking (with this new variable) there is at least another vaul token to SwitchVaultCollateral

## Impact Details

If all current collateral vault token are deprecated and vault wont have options to SwitchVaultCollateral and will become irremediably liquidatable

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralTypes.sol#L14-L19\\>
<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralTypes.sol#L63-L75>

## Proof of Concept

## Proof of Concept

AgentVault is created with vault collateral token T\
All vault collateral tokens (including T) are deprecated with CollateralTypes.sol::deprecate\
AgentVault is unable to SwitchVaultCollateral and will be eventually liquidated
