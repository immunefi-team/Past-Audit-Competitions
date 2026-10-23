# 52393 sc low burns blocked by both sides whitelist with zero address exclusion when restrictions are enabled

**Submitted on Aug 10th 2025 at 12:08:27 UTC by @Khay3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52393
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

### Summary

When `transfersAllowed == false`, `WhitelistRestrictions::isTransferAllowed` requires both `from` and `to` to be whitelisted. Because the zero address cannot be whitelisted, any burn (`to == address(0)`) is permanently disallowed under restriction. This blocks protocol flows that depend on burning (supply reduction, redemption, bridging, fee-burning) whenever restrictions are active.

### Vulnerability Detail

The `isTransferAllowed` function enforces a strict “both-sides must be whitelisted” rule under restriction:

[WhitelistRestrictions::isTransferAllowed()](https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol):

```solidity
function isTransferAllowed(address from, address to, uint256 /*amount*/ ) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();

    // If transfers are unrestricted, allow all transfers
    if (ws.transfersAllowed) {
        return true;
    }

    // Otherwise, only allow if both the sender and receiver are whitelisted
    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

The whitelist admin/manager cannot add the zero address, making `to == address(0)` (burn) impossible under restriction:

[WhitelistRestrictions::addToWhitelist()](https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol):

```solidity
function addToWhitelist(address account) external onlyRole(MANAGER_ROLE) {
    if (account == address(0)) {
        revert InvalidAddress();
    }
    ...
}
```

The batch variant also refuses to add zero (silently skipping it):

[WhitelistRestrictions::batchAddToWhitelist()](https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol):

```solidity
for (uint256 i = 0; i < accounts.length; i++) {
    address account = accounts[i];
    if (account == address(0)) {
        continue; // Skip zero address
    }
    ...
}
```

In short: `isTransferAllowed` enforces both `from` and `to` whitelisted while the zero address cannot be whitelisted, therefore any burn to `address(0)` can never satisfy the condition when restrictions are enabled.

### Impact

Burns are impossible whenever restrictions are active. This prevents supply reduction, fee-burning, redemption flows that rely on burn-to-zero, and common bridge patterns that require token burns on the origin chain. Operations such as emergency redemptions or compliance-driven supply adjustments are blocked, potentially freezing required lifecycle actions and creating governance and operational risks.

If burn gating is not intended, users and integrators will experience unexplained failures in legitimate burn calls during periods when restrictions are toggled on, degrading protocol reliability.

## Recommendation

{% hint style="info" %}

* Special-case the zero address in `isTransferAllowed`:
  * If `to == address(0)` (burn), consider allowing the burn when `from` is whitelisted (or when the caller has `ADMIN_ROLE`/`MANAGER_ROLE`).
  * If `from == address(0)` (mint), likewise allow when `to` is whitelisted (or authorized).
* Optionally make the rule configurable:
  * Add a mode switch (both-sides vs one-side whitelist) and explicit mint/burn exemptions.
* Keep zero address non-whitelistable; handle exemptions in logic instead:
  * Example logic:
    * If restricted and `to == address(0)`: return `ws.isWhitelisted[from]` (or role-gated).
    * If restricted and `from == address(0)`: return `ws.isWhitelisted[to]` (or role-gated).
      {% endhint %}

## Proof of Concept

<details>

<summary>High level PoC: burn blocked under restriction (click to expand)</summary>

Step-by-step POC (burn blocked under restriction):

StepAdmin sets WhitelistRestrictions::setTransfersAllowed(false).StepA token integrated with ITransferRestrictions attempts to burn: from = user, to = address(0).StepisTransferAllowed(user, address(0), amount) returns ws.isWhitelisted\[user] && ws.isWhitelisted\[address(0)].Stepws.isWhitelisted\[address(0)] is always false; zero cannot be added.StepAuthorization fails; the token’s burn pathway reverts.

Conditions to allow the issue:

* `WhitelistRestrictions::setTransfersAllowed(false)` (restriction enabled).
* Token integrates and enforces `ITransferRestrictions::isTransferAllowed` on burn.
* Zero address cannot be whitelisted (`addToWhitelist` rejects, `batchAddToWhitelist` skips).

</details>
