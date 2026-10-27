# 52218 sc high creator retains default admin role allowing bypass of upgrade restrictions

**Submitted on Aug 8th 2025 at 18:45:49 UTC by @aksoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52218
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Token Creator Can Upgrade To Malicous Implementation

## Description

### Brief/Intro

The ArcTokenFactory is designed to restrict token upgrades to only whitelisted implementations to prevent malicious token creators from injecting harmful logic. However, due to flawed access control in the upgrade mechanism, creators can still directly upgrade their ArcToken contracts without going through the factory’s checks. This completely bypasses the intended restriction.

### Vulnerability Details

The ArcTokenFactory enforces upgrade safety by ensuring only pre-approved (whitelisted) implementation addresses can be used for upgrades. This mechanism is intended to mitigate the risk of token creators adding malicious code after deployment. When creating a token, the upgrader role is assigned to the factory to prevent the creator from upgrading token .

```solidity
  function createToken(...
    ) external returns (address) {
      ...

        // Grant all necessary roles to the owner
        // Grant the DEFAULT_ADMIN_ROLE to the deployer
       ...
        token.grantRole(token.UPGRADER_ROLE(), address(this));
```

The token contract uses OpenZeppelin’s AccessControl for role management. While upgradeability is intended to be restricted (e.g., via a global whitelist or controlled governance process), the creator of the token retains the DEFAULT\_ADMIN\_ROLE after deployment.

In OpenZeppelin’s access model, DEFAULT\_ADMIN\_ROLE has full authority over all roles, including the ability to grant or revoke the UPGRADER\_ROLE. This means that even if the creator is not initially assigned the upgrader role, they can later grant it to themselves or to any other address they control. By doing so, the creator can upgrade the token contract to any arbitrary implementation, potentially bypassing global whitelist checks or other upgrade restrictions in the protocol.

### Impact Details

The creator can bypass global upgrade controls and upgrade the token.

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L193>

## Proof of Concept

{% stepper %}
{% step %}

### Step

The token creator deploys the contract and retains the DEFAULT\_ADMIN\_ROLE, which allows granting or revoking any role in the contract.
{% endstep %}

{% step %}

### Step

At any point in the future, the creator decides to grant the UPGRADER\_ROLE to themselves.
{% endstep %}

{% step %}

### Step

With the UPGRADER\_ROLE, the creator can upgrade the token contract implementation to any arbitrary contract.
{% endstep %}

{% step %}

### Step

The new implementation can contain malicious logic, such as:

* Locking all users’ tokens to prevent sales or transfers.
* Bypassing the protocol’s global whitelist checks.
* Allowing blacklisted addresses to transfer tokens or earn yield.
  {% endstep %}

{% step %}

### Step

As a result, the upgrade could be used to bypass protocol restrictions, or perform malicious actions without any checks from the original deployment.
{% endstep %}
{% endstepper %}
