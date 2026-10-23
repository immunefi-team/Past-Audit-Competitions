# #45439 \[SC-Low] Empty String Allowed as Pool Token Suffix in \_reserveAndValidatePoolTokenSuffix

**Submitted on May 14th 2025 at 17:52:31 UTC by @EFCCWEB3 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45439
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `_reserveAndValidatePoolTokenSuffix` function, called by `createAgentVault` in the FAssets system, allows an empty string ("") as a valid `poolTokenSuffix`, lacking a minimum length check. Although the FAssets Agent CLI validates tmp.agent-settings.json with a regex `(^[A-Z0-9](?:[A-Z0-9\\-]{0,18}[A-Z0-9])?$)` to prevent empty suffixes, the contract’s vulnerability persists, enabling invalid pool token symbols (e.g., FCPT-XRP-) via direct contract calls or CLI bypasses. This risks token conflicts, exchange/wallet rejections, and operational disruptions on mainnet, potentially locking millions in funds.

## Vulnerability Details

The `_reserveAndValidatePoolTokenSuffix` function validates the poolTokenSuffix for FAsset Collateral Pool Tokens, set in `tmp.agent-settings.json` during agent creation. The suffix forms part of the token symbol `(e.g., FCPT-XRP-MY-ALPHA-AGENT-1)`. The CLI enforces a regex `(^[A-Z0-9](?:[A-Z0-9\\-]{0,18}[A-Z0-9])?$)`, requiring at least one character, but the contract does not, allowing empty strings. The vulnerability is triggered in `createAgentVault`:

```solidity
function createAgentVault(
    IIAssetManager _assetManager,
    IAddressValidity.Proof calldata _addressProof,
    AgentSettings.Data calldata _settings
) internal returns (address) {
    AssetManagerState.State storage state = AssetManagerState.get();
    _reserveAndValidatePoolTokenSuffix(_settings.poolTokenSuffix);
    // ... creates agent vault
}
```

```solidity
function _reserveAndValidatePoolTokenSuffix(string memory _suffix)
    private
{
    AssetManagerState.State storage state = AssetManagerState.get();
    require(!state.reservedPoolTokenSuffixes[_suffix], "suffix already reserved");
    state.reservedPoolTokenSuffixes[_suffix] = true;
    bytes memory suffixb = bytes(_suffix);
    uint256 len = suffixb.length;
    require(len < MAX_SUFFIX_LEN, "suffix too long");
    for (uint256 i = 0; i < len; i++) {
        bytes1 ch = suffixb[i];
        require((ch >= "A" && ch <= "Z") || (ch >= "0" && ch <= "9") || (i > 0 && i < len - 1 && ch == "-"),
            "invalid character in suffix");
    }
}
```

.

## Impact Details

Contract fails to deliver promised returns, but doesn't lose value

## Recommendation

Add any relevant links to documentation or code

```solidity
function _reserveAndValidatePoolTokenSuffix(string memory _suffix)
    private
{
    AssetManagerState.State storage state = AssetManagerState.get();
    require(!state.reservedPoolTokenSuffixes[_suffix], "suffix already reserved");
    state.reservedPoolTokenSuffixes[_suffix] = true;
    bytes memory suffixb = bytes(_suffix);
    uint256 len = suffixb.length;
++    require(len > 0, "suffix cannot be empty");
    require(len < MAX_SUFFIX_LEN, "suffix too long");
    for (uint256 i = 0; i < len; i++) {
        bytes1 ch = suffixb[i];
        require((ch >= "A" && ch <= "Z") || (ch >= "0" && ch <= "9") || (i > 0 && i < len - 1 && ch == "-"),
            "invalid character in suffix");
    }
}
```

## Proof of Concept

## Proof of Concept

* User sets poolTokenSuffix = `""` in `tmp.agent-settings.json`.
* CLI command (agent-bot create) passes the empty suffix to `createAgentVault`.

`_reserveAndValidatePoolTokenSuffix("")` accepts the suffix, creating an agent with token symbol `FCPT-XRP-`.
