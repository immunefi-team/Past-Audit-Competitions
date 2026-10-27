# 52221 sc insight hardcoded supra subscription wallet can freeze spin

* Submitted on Aug 8th 2025 at 19:47:07 UTC by @holydevoti0n for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network)
* Report ID: #52221
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* Impacts:
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief / Intro

The `admin` address in the `Spin` contract is immutable and used as `_clientWalletAddress` for all Supra dVRF requests. If it ever needs to be changed, the contract can’t update it, so future `generateRequest` calls revert because the fixed address must stay whitelisted and funded.

### Vulnerability Details

In `generateRequest` within the `Spin` contract the `admin` value is passed as the `_clientWalletAddress`:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L190>

```solidity
// immutably set at initialise()
address public admin;                            // ← subscription wallet
...
uint256 nonce = supraRouter.generateRequest(
        callbackSignature,
        rngCount,
        numConfirmations,
        clientSeed,
        admin                     // ← _clientWalletAddress
);
```

Per Supra dVRF docs: <https://docs.supra.com/dvrf/v2-guide>

* `_clientWalletAddress` must be “the client wallet address that is already registered with the Supra team”, i.e. the subscription owner that whitelists consumer contracts and funds the callback deposit.

The contract sets `admin` to `msg.sender` in `initialize()` and provides no setter. Because `admin` is effectively immutable after initialization, rotating the on-chain client wallet (the Supra subscription wallet) is impossible. If the external Supra subscription is switched to a different wallet, calls that use the hardcoded `admin` will revert.

### Impact Details

Spins become permanently impossible as soon as the original Supra subscription (`admin`) wallet must be replaced with a new whitelisted and funded wallet.

## Recommendation

Provide a governance-controlled function to update the `admin` address so the contract can use a new client wallet address in `generateRequest`. Ensure this setter is access-controlled (e.g., onlyOwner / onlyGovernance) and consider emitting an event on change.

{% hint style="warning" %}
Make sure the new setter is protected by the appropriate access control and that change-of-admin processes are coordinated with Supra (so the new address is whitelisted and funded before switching).
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Step

Once initialized, the `Spin` contract stores wallet A as `admin`.
{% endstep %}

{% step %}

### Step

Team registers and whitelists wallet A with Supra.
{% endstep %}

{% step %}

### Step

Rotation is needed (wallet A compromised or retired); team switches subscription to wallet B on Supra.
{% endstep %}

{% step %}

### Step

`Spin` contract still sends requests using wallet A (cannot change `admin`). Supra rejects the request because A is no longer valid, causing `generateRequest` to revert and spins to fail.
{% endstep %}
{% endstepper %}
