# 50187 sc insight yieldblacklistrestrictions uses slot 0 instead of unstructured storage risking slot collision

**Submitted on Jul 22nd 2025 at 11:32:04 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50187
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/YieldBlacklistRestrictions.sol>

{% hint style="info" %}
Category: Insight → Security best practices
{% endhint %}

## Description

### Vulnerability Details

Contract `YieldBlacklistRestrictions` declares its `_isBlacklisted` mapping in the compiler-assigned slot 0 rather than behind its own unstructured storage slot.

This deviates from the pattern used by contract `WhitelistRestrictions` and from standard upgradeable contract practices. It opens the door to storage collisions with other inherited contracts or future module upgrades, potentially corrupting the blacklist and allowing unauthorized yield distributions.

By not allocating a unique slot for the blacklist mapping, any additional state added in parent contracts or future upgrades could land in slot 0, overwriting `_isBlacklisted` data or vice versa.

### Impact Details

Potential High Impact if Exploited:

* Theft of Unclaimed Yield: Corrupted blacklist entries may unintentionally whitelist addresses that should be blocked, allowing them to claim yield.
* Denial of Service: Overwriting blacklist state could render the contract unable to enforce restrictions, freezing or disabling legitimate yield distributions.

Although currently each module is deployed behind its own proxy, the inconsistent storage pattern is a latent risk that can manifest during upgrades, inheritance changes, or multi-module compositions.

## Proof of Concept (code examples)

<details>

<summary>Proof of Concept — direct mapping declaration vs. isolated slot</summary>

In `YieldBlacklistRestrictions.sol` the blacklist state is declared directly as:

```solidity
    // Mapping for yield blacklist (address => true if blacklisted)
    mapping(address => bool) private _isBlacklisted;
```

Contrastingly, `WhitelistRestrictions` isolates its state in a dedicated slot via:

```solidity
bytes32 private constant WHITELIST_STORAGE_LOCATION = keccak256("whitelist.restrictions.storage");
```

</details>

## Recommended Mitigation (best practice)

* Move the `_isBlacklisted` mapping behind an unstructured storage slot (use a unique `bytes32` constant derived from a descriptive string and access it via inline assembly or a storage struct pattern).
* Follow the same storage-slot isolation pattern used in `WhitelistRestrictions` and other upgradeable modules to prevent future collisions.
* Where practical, add tests to ensure that adding state variables in parent/child contracts does not change the layout of existing storage used by modules.

## References

* Target source: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/YieldBlacklistRestrictions.sol>
* Related pattern example (WhitelistRestrictions): uses `keccak256` constant for storage location.
