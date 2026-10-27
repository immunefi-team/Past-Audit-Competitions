# 53048 sc medium approval logic can break on non standard erc 20s usdt style and leave allowances loose

Submitted on Aug 14th 2025 at 18:05:29 UTC by @jpmendes for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #53048
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>

## Brief / Intro

{% hint style="warning" %}
The contract uses direct `safeApprove` calls without resetting to zero first. This can both break deposits for non-standard ERC-20 tokens (like USDT) and leave residual allowances that a malicious or compromised spender could later exploit to drain tokens directly from the wrapper contract.
{% endhint %}

## Vulnerability Details

The following allowance updates are performed without first zeroing the current allowance:

```solidity
depositAsset.safeApprove(address(aggregator), depositAmount);
depositAsset.safeApprove(okxApprover, fromTokenAmount);
supportedAsset.safeApprove(vaultAddress, supportedAssetAmount);
```

* Some ERC-20 tokens (e.g., USDT) revert when updating an allowance directly from a non-zero value to another non-zero value; they require the allowance be set to zero first. This will cause deposits to revert for those tokens.
* If the approved spender (router/approver) spends less than the approved amount, the remaining allowance remains. If that spender is later compromised or malicious, they can call transferFrom to withdraw the leftover tokens from the wrapper contract.

## Impact

{% hint style="danger" %}

* Direct theft: Leftover allowances can be exploited by a malicious/compromised spender to steal tokens directly from the contract after a partial fill.
* Permanent DoS for certain assets: Deposits with zero-first-approval tokens (e.g., USDT) will fail, effectively blocking those assets.
  {% endhint %}

## Proof of Concept

Scenario: a user deposits 100 tokens via the 1inch path. The 1inch aggregator only needs 50 tokens for the swap (partial fill). Because the contract did not reset allowance, 50 tokens remain approved and can be stolen later.

{% stepper %}
{% step %}

### User deposit and partial fill

1. Contract approves 100 tokens to the aggregator:

```solidity
depositAsset.safeApprove(aggregator, 100);
```

2. Aggregator.swap() uses 50 tokens to perform the swap (partial fill). 50 tokens remain in the wrapper contract and are still approved for the aggregator.
   {% endstep %}

{% step %}

### Attacker later drains leftover allowance

1. Attacker (or compromised aggregator) calls:

```solidity
depositAsset.transferFrom(wrapperAddress, attackerAddress, 50);
```

2. Attacker receives 50 tokens without interacting with vault share logic.
   {% endstep %}
   {% endstepper %}

## References

<details>

<summary>Relevant references</summary>

* EIP-20: <https://eips.ethereum.org/EIPS/eip-20>
* USDT allowance behavior (source contract): <https://etherscan.io/address/0xdAC17F958D2ee523a2206206994597C13D831ec7#code>

</details>
