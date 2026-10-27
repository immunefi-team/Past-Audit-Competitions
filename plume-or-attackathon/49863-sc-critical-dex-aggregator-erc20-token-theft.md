# 49863 sc critical dex aggregator erc20 token theft

**Submitted on Jul 20th 2025 at 02:53:53 UTC by @Blobism for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49863
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

{% hint style="danger" %}
Critical: An attacker can grant themselves the ability to spend any ERC20 token that the contract holds by invoking OKX methods with a malicious teller contract and specific OKX swap parameters.
{% endhint %}

## Description

### Brief / Intro

An attacker can grant themselves the ability to spend any ERC20 token that the contract holds by invoking OKX methods with a malicious teller contract and specific OKX swap parameters.

### Vulnerability Details

The key issue being exploited is that `_okxHelper` does not validate that the swap receiver is the current contract. This contrasts with `_oneInchHelper`, which reverts if `desc.dstReceiver != address(this)`.

An attacker can route an OKX swap so the swapped tokens go directly to an address they control, then leverage the contract's subsequent approval logic to approve a vault they control to spend the ERC20 token the contract still holds. If the contract currently holds that token, the attacker can effectively get double the amount they swapped to.

Vulnerable excerpt from `_okxHelper`:

```solidity
// <----------- BUG: no check that the swap receiver is the current contract
// Execute the swap with the provided calldata
(bool success, bytes memory result) = address(okxRouter).call(okxCallData);
if (!success) {
    assembly {
        revert(add(result, 32), mload(result))
    }
}

// Decode the return value
supportedAssetAmount = abi.decode(result, (uint256));

// Approve teller's vault to spend the supported asset
// Cast teller address to TellerWithMultiAssetSupport to call vault()
address vaultAddress = address(TellerWithMultiAssetSupport(payable(teller)).vault());
// ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
// ISSUE: teller is passed in by the user, meaning attacker controls the vault

if (vaultAddress == address(0)) {
    revert("DexAggregatorWrapper: Invalid vault address for approval");
}
// Use standard approve (as requested)
supportedAsset.safeApprove(vaultAddress, supportedAssetAmount);
// ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
// attacker vault gets approved to steal ERC20 from the contract

// Return value needs to be here since it's declared in the function signature
return supportedAssetAmount;
```

How the attacker invokes the vulnerable flow (example using `depositOkxUniversal`):

```solidity
function depositOkxUniversal(
    // ...
)
    // ...
{
    _checkPredicateProxy(predicateMessage);
    uint256 supportedAssetAmount =
        _okxHelper(supportedAsset, address(teller), fromToken, fromTokenAmount, okxCallData, nativeValueToWrap);

    // Deposit assets
    shares = teller.deposit(supportedAsset, supportedAssetAmount, minimumMint);
    // ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // attacker controlled teller does a fake deposit

    // Get vault address
    address vaultAddress = address(teller.vault());
    if (vaultAddress == address(0)) {
        revert("DexAggregatorWrapper: Invalid vault address");
    }
    // Use safeTransfer to send shares to msg.sender
    ERC20(vaultAddress).safeTransfer(msg.sender, shares);
    _calcSharesAndEmitEvent(
        supportedAsset, CrossChainTellerBase(address(teller)), fromToken, fromTokenAmount, supportedAssetAmount
    );
}
```

### Impact Details

The attack requires initial funds (flash loans possible) and an ERC20 token currently held by the contract. Leftover tokens in the contract from DEX swaps are possible (note: a distinct bug #49854 can result in tokens being left in the contract), making this vulnerability exploitable to steal those tokens.

### References

See `src/helper/DexAggregatorWrapperWithPredicateProxy.sol` in the target repository: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>

## Proof of Concept

{% stepper %}
{% step %}

### Step

Attacker sees there is some WETH left in the aggregator contract.
{% endstep %}

{% step %}

### Step

Attacker approves `DexAggregatorWrapperWithPredicateProxy` to spend some of their USDC (they will receive the value back as WETH from their swap, plus the WETH they steal).
{% endstep %}

{% step %}

### Step

Attacker deploys a fake `TellerWithMultiAssetSupport` which:

* Contains a vault address they control.
* Does not revert when called by the aggregator contract.
* Contains a `deposit` method which does not actually perform the deposit (fake deposit).
  {% endstep %}

{% step %}

### Step

Attacker calls `depositOkxUniversal`, passing in their fake teller, and sets `okxCallData` such that the `USDC->WETH` swap routes the swapped WETH to an address they own (so they regain the original funds).
{% endstep %}

{% step %}

### Step

The aggregator contract approves the malicious vault to transfer an amount of WETH equal to what was returned from the swap (approved via `supportedAsset.safeApprove(vaultAddress, supportedAssetAmount)`).
{% endstep %}

{% step %}

### Step

The attacker uses their vault to transfer the WETH from the contract to themselves, effectively stealing the contract-held WETH and doubling their initial investment.
{% endstep %}
{% endstepper %}
