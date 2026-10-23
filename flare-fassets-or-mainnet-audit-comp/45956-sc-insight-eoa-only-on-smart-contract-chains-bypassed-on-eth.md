# #45956 \[SC-Insight] EOA only on smart contract chains bypassed on ETH

**Submitted on May 22nd 2025 at 21:58:14 UTC by @Machicoulis for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45956
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/AgentsCreateDestroy.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

With EIP-7702 enabling EOAs to temporarily delegate to smart contracts, agents can now implement discriminatory logic during minting, accepting payments only from whitelisted addresses while rejecting others. This breaks the permissionless nature of the FAsset system and allows agents to create unfair markets, potentially excluding users based on arbitrary criteria.

## Vulnerability Details

The vulnerability allows agents to implement conditional acceptance of minting payments using EIP-7702 delegation.

When users attempt to mint: User calls reserveCollateral() and pays collateral reservation fee.

User receives payment instructions via CollateralReserved event.

User sends payment to agent's underlying address.

Agent's EOA (delegated to discriminatory contract) rejects the payment

User loses collateral reservation fee with no recourse

The protocol's EOA requirement was specifically designed to prevent such conditional behavior:

```
// From AgentsCreateDestroy.sol
require(Globals.getSettings().requireEOAAddressProof, "EOA proof required");
```

## Impact Details

Because of all the other safeguards implemented, even if the agent bypasses the EOA only security on Ethereum, they can't do very harmful things. This is mostly a griefing attack, and a break of an invariant that should be aknowledged.

## References

<https://eips.ethereum.org/EIPS/eip-7702>

## Proof of Concept

## Proof of Concept

Agent deploys a smart contract on Ethereum that only accepts payments ≥ 10 ETH Agent creates vault on Flare by calling createAgentVault() with their Ethereum address Agent joins available agents list by calling makeAgentAvailable()

Small user (Alice) wants to mint 0.5 ETH worth of f-assets Alice calls reserveCollateral() on Flare, paying 0.001 ETH collateral reservation fee Alice receives payment instructions via CollateralReserved event: send 0.505 ETH to agent's Ethereum address

Alice sends 0.505 ETH to agent's Ethereum address Agent uses EIP-7702 to delegate their EOA to the minimum deposit contract for this transaction Transaction reverts because 0.505 ETH < 10 ETH minimum Alice gets her 0.505 ETH back but has lost the collateral reservation fee

Alice cannot call executeMinting() - she has no payment proof (payment failed) Agent calls mintingPaymentDefault() after timeout and collects Alice's reservation fee Agent continues normal operations with large depositors while excluding small ones
