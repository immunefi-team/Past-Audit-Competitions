# 57426 sc medium dynamic price signature replay allows unlimited minting at historical prices

**Submitted on Oct 26th 2025 at 04:36:36 UTC by @iamephraim for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57426
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:**
  * Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

## Description

### Brief/Intro

Users can replay old dynamic price signatures to mint NFTs at previously signed prices and thereby bypass current contract pricing.

### Vulnerability Details

The `mintDynamicPrice` function validates signatures containing historical prices but uses those same prices for payment calculation. When collection owners increase prices via the `setNftParameters()` function, users with old signatures can repeatedly mint at the original lower prices, causing the owner to lose funds on their NFTs.

```solidity
function mintDynamicPrice(...) expectedTokenCheck(..) external payable nonReentrant {
    // ...
    for (uint256 i; i < paramsArray.length; ++i) {
        factoryParameters.signerAddress.checkDynamicPriceParameters(receiver, paramsArray[i]); //validate signature with old price
        unchecked {
            amountToPay += paramsArray[i].price; // Sums up old prices
        }

        _baseMint(paramsArray[i].tokenId, receiver, paramsArray[i].tokenUri);
    }
    
    _pay(amountToPay, expectedPayingToken); // Pay with the old prices
}
```

Location: contracts/v2/tokens/AccessToken.sol:209-223

The signature validation accepts historical prices without verifying they match current contract pricing, allowing replay attacks that bypass price updates.

This is a common issue in the nft.cairo::mintDynamicPrice function which is also in scope.

### Impact Details

This issue enables users to mint unlimited NFTs at discounted historical prices, causing significant revenue loss for collection owners and undermining the intended pricing strategy of the NFT collection.

### Recommendation

Add price validation to ensure signature prices match current contract prices, or implement a nonce system to prevent signature replay attacks.

```solidity
// Add validation in mintDynamicPrice
require(paramsArray[i].price == info.mintPrice, "Price mismatch");
```

{% hint style="warning" %}
Ensure the contract compares the signed price to the current on-chain price (or uses a nonce/expiry tied to specific parameters) before accepting the signature. This prevents old signed prices from being reused after a price change.
{% endhint %}

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L209>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/SignatureVerifier.sol#L221-L232>
* <https://github.com/immunefi-team/audit-comp-belong/blob/0cbcde6fd80dbc55a9e3403c8e5a74827dea19e2/src/nft/nft.cairo#L279>

## Link to Proof of Concept

<https://gist.github.com/Ephraim-nonso/24e4cc2f407dfa7a656da7f75e4df1f7>

## Proof of Concept

{% stepper %}
{% step %}

### Initial signature acquisition

* The NFT collection is initially priced at 0.01 ETH per mint.
* A user requests a signature from the platform signer to mint an NFT at the current price of 0.01 ETH.
* The signer creates a signature that includes: (userAddress, tokenId, tokenUri, 0.01 ETH, chainId).
* This signature is valid for minting at exactly 0.01 ETH.
  {% endstep %}

{% step %}

### Successful initial mint

* The user successfully mints their first NFT using the signature.
* They pay exactly 0.01 ETH as expected.
* The transaction completes normally, and the user receives their NFT.
  {% endstep %}

{% step %}

### Price increase by collection owner

* The collection owner decides to increase the mint price from 0.01 ETH to 0.02 ETH.
* The owner calls setNftParameters() to update the contract's mint price.
* All future mints should now cost 0.02 ETH instead of 0.01 ETH.
  {% endstep %}

{% step %}

### The attack — reusing old signature

* The user still has their old signature that was created when the price was 0.01 ETH.
* Instead of getting a new signature for the new 0.02 ETH price, they reuse their old signature.
* They call mintDynamicPrice() with:
  * The same old signature
  * A different token ID (to avoid minting the same token twice)
  * The price field set to 0.01 ETH (from the old signature)
  * Only 0.01 ETH sent as payment
    {% endstep %}

{% step %}

### Attack succeeds

* The contract validates the signature and finds it's valid (because it was signed for 0.01 ETH).
* The contract uses the price from the signature (0.01 ETH) instead of checking against the current contract price (0.02 ETH).
* The user successfully mints another NFT while only paying 0.01 ETH.
* The collection owner receives only 0.01 ETH instead of the intended 0.02 ETH.
  {% endstep %}

{% step %}

### Repeated attacks

* The user can continue this attack indefinitely.
* They can mint as many NFTs as they want using the same old signature.
* Each time, they only pay 0.01 ETH instead of the current 0.02 ETH price.
* The collection owner loses 0.01 ETH on every mint (50% revenue loss).
  {% endstep %}
  {% endstepper %}
