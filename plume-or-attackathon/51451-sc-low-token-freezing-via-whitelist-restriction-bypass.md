# 51451 sc low token freezing via whitelist restriction bypass

**Submitted on Aug 2nd 2025 at 23:27:29 UTC by @technicalattri for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #51451
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

The `WhitelistRestrictions` contract does not account for the zero address (`address(0)`) when `transfersAllowed` is set to `false`. As a result, minting and burning tokens becomes permanently blocked, even when user addresses are correctly whitelisted.

Impact: An attacker can exploit this vulnerability to permanently freeze core token functionality by bypassing whitelist restrictions on the zero address. Once transfer restrictions are enabled by the admin, any token operation involving the zero address (such as minting from `address(0)` or burning to `address(0)`) fails because the zero address is never whitelisted due to an intentional `continue` statement in `batchAddToWhitelist()`. Therefore, minting new tokens or burning existing ones becomes impossible, halting token supply adjustments and potentially rendering the token ecosystem economically unusable.

## Recommendation

Update `isTransferAllowed` to handle mint and burn cases explicitly, instead of blindly requiring both `from` and `to` to be whitelisted:

{% code title="Suggested fix" %}

```solidity
function isTransferAllowed(address from, address to, uint256) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();
    if (ws.transfersAllowed) return true;

    // Handle minting and burning
    if (from == address(0)) return ws.isWhitelisted[to]; // Minting
    if (to == address(0)) return ws.isWhitelisted[from]; // Burning

    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

{% endcode %}

This ensures mint and burn operations depend only on the user being whitelisted, not `address(0)`.

## Proof of Concept

Admin enables transfer restrictions:

```solidity
setTransfersAllowed(false);
```

{% stepper %}
{% step %}

### Attack step

Attempt to whitelist multiple addresses:

```solidity
batchAddToWhitelist([0xAAA, 0xBBB, address(0)]);
```

The function silently skips the zero address:

```solidity
if (account == address(0)) {
    continue;
}
```

As a result, `address(0)` is never whitelisted.
{% endstep %}

{% step %}

### Consequence

`address(0)` is critical for token supply operations:

* Minting: `from == address(0)`
* Burning: `to == address(0)`

Because the current logic requires both parties to be whitelisted, mint/burn transfers are blocked.
{% endstep %}

{% step %}

### Example failing check

When ArcToken attempts to burn tokens:

```solidity
isTransferAllowed(user, address(0), amount) // → returns false
```

Since `address(0)` is never whitelisted, the call fails, making minting/burning impossible.
{% endstep %}
{% endstepper %}
