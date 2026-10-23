# #55241 \[SC-Insight] insufficient validation of pool token suffix allows consecutive hyphens enables token symbol impersonation and user confusion

**Submitted on Sep 25th 2025 at 08:52:10 UTC by @r1ver for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* **Report ID:** #55241
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/commit/59373cee12e6d2a9fa0a9cc8735bb486faa51b36>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

Pool token suffix validation is incomplete. Although empty strings are disallowed and characters are restricted to A–Z, 0–9, and hyphen, the logic permits consecutive/stacked hyphens within the agent suffix. This allows creation of token names/symbols that closely impersonate legitimate ones (e.g., `FCPT-BTC-AG-X` vs `FCPT-BTC-AG--X`), leading to UI/wallet/exchange confusion and impersonation risk. In production, this can cause misidentification, mistaken transfers, listing issues, and operational disruptions.

## Vulnerability Details

The agent-provided pool token suffix is validated character-by-character to allow uppercase letters, digits, and hyphens, but the check only forbids hyphens at the first and last positions. It does not prevent adjacent hyphens, so values like `AG--X` or `A---B` pass validation. This arises from the predicate that permits a hyphen whenever the index is strictly inside the string, without enforcing non-adjacency or a limit on the number of hyphens.

Relevant validation snippet:

```solidity
247:256:fassets/contracts/assetManager/library/AgentsCreateDestroy.sol
// validate - require only printable ASCII characters (no spaces) and limited length
bytes memory suffixb = bytes(_suffix);
uint256 len = suffixb.length;
require(len >= MIN_SUFFIX_LEN, "suffix too short");
require(len < MAX_SUFFIX_LEN, "suffix too long");
for (uint256 i = 0; i < len; i++) {
    bytes1 ch = suffixb[i];
    // allow A-Z, 0-9 and '-' (but not at start or end)
    require((ch >= "A" && ch <= "Z") || (ch >= "0" && ch <= "9") || (i > 0 && i < len - 1 && ch == "-"),
        "invalid character in suffix");
}
```

When creating pool tokens, the factory concatenates a hyphen and the agent suffix into both the token name and symbol. Because the agent suffix can contain consecutive hyphens and there is no normalization at creation time, these sequences directly propagate to the on-chain identifiers.

```solidity
20:29:fassets/contracts/assetManager/implementation/CollateralPoolTokenFactory.sol
function create(IICollateralPool _pool, string memory _systemSuffix, string memory _agentSuffix)
    external override
    returns (address)
{
    string memory tokenName = string.concat(TOKEN_NAME_PREFIX, _systemSuffix, "-", _agentSuffix);
    string memory tokenSymbol = string.concat(TOKEN_SYMBOL_PREFIX, _systemSuffix, "-", _agentSuffix);
    ERC1967Proxy proxy = new ERC1967Proxy(implementation, new bytes(0));
    CollateralPoolToken poolToken = CollateralPoolToken(address(proxy));
    poolToken.initialize(address(_pool), tokenName, tokenSymbol);
    return address(poolToken);
}
```

## Impact Details

Allowing consecutive hyphens enables near-duplicate pool token symbols (e.g., `FCPT-BTC-AG-X` vs `FCPT-BTC-AG--X`) that many UIs/wallets/exchanges render as indistinguishable, causing users to misidentify and buy/deposit the wrong asset, resulting in direct value loss. It also fragments liquidity and distorts pricing, and may trigger listing/rejection issues, aligning with impersonation/griefing and “contract fails to deliver promised returns, but doesn’t lose value” impacts.

## References

* <https://github.com/flare-foundation/fassets/blob/59373cee12e6d2a9fa0a9cc8735bb486faa51b36/contracts/assetManager/library/AgentsCreateDestroy.sol#L247-L256>
* <https://github.com/flare-foundation/fassets/blob/59373cee12e6d2a9fa0a9cc8735bb486faa51b36/contracts/assetManager/implementation/CollateralPoolFactory.sol#L20-L29>

## Proof of Concept

{% stepper %}
{% step %}

### Step

User sets `poolTokenSuffix = "AG--X"` in `tmp.agent-settings.json`.
{% endstep %}

{% step %}

### Step

Run agent creation (`agent-bot create`) which forwards the suffix to `createAgentVault`.
{% endstep %}

{% step %}

### Step

`_reserveAndValidatePoolTokenSuffix("AG--X")` accepts; a pool token is created with symbol `FCPT-BTC-AG--X`.
{% endstep %}
{% endstepper %}
