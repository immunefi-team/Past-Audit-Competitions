# 57895 sc medium lack of msg sender validation in collection creation signature enables front running attack leading to creator impersonation

**Submitted on Oct 29th 2025 at 11:51:25 UTC by @Yuubee for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57895
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Unauthorized minting of NFTs

***

## Description

### Brief/Intro

The `Factory.produce()` function uses signature verification to authorize AccessToken collection creation, but the signature does not include the intended creator's address (`msg.sender`). This allows an attacker to intercept a valid signature and front-run the legitimate transaction, deploying the collection under their own address and receiving all mint revenues intended for the original creator.

***

## Vulnerability Details

In `SignatureVerifier.sol`, the `checkAccessTokenInfo` function validates collection creation without binding the signature to a specific creator:

{% code title="SignatureVerifier.sol:checkAccessTokenInfo" %}

```solidity
function checkAccessTokenInfo(address signer, AccessTokenInfo memory accessTokenInfo) external view {
    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    block.chainid
                )
            ),
            accessTokenInfo.signature
        ),
        InvalidSignature()
    );
}
```

{% endcode %}

Notice that `msg.sender` is **NOT** included in the signed message.

In `Factory.sol:produce()`, the creator is set based on the transaction sender, not validated against the signature:

{% code title="Factory.sol:produce (relevant excerpt)" %}

```solidity
function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode)
    external
    returns (address nftAddress)
{
    // ... signature check (doesn't validate msg.sender) ...
    
    AccessToken(nftAddress).initialize(
        AccessToken.AccessTokenParameters({
            factory: Factory(address(this)),
            info: accessTokenInfo,
            creator: msg.sender,  // <-- Attacker becomes creator!
            feeReceiver: receiver,
            referralCode: referralCode
        }),
        factoryParameters.transferValidator
    );
}
```

{% endcode %}

Because the signature doesn't bind to the expected creator address, anyone who obtains a valid signature and parameters can call `produce()` and become the creator.

***

## Impact Details

An attacker monitoring the mempool can:

* Intercept legitimate collection creation transactions containing valid backend signatures
* Extract the signature and parameters from the pending transaction
* Submit their own transaction with higher gas to front-run the legitimate transaction
* Deploy the collection under their address, becoming the creator
* Receive all mint revenues that should go to the legitimate creator
* Prevent the legitimate creator from deploying a collection with that name/symbol (due to the uniqueness check)

Consequences:

* Direct financial loss to legitimate creators (all mint fees)
* Brand/collection hijacking if the name/symbol has value
* Denial of service for the intended creator
* Reputational damage to the platform

***

{% hint style="warning" %}
This vulnerability enables creator impersonation and financial theft via a mempool front-running attack. It should be considered a high-priority fix for any production deployment that issues off-chain signatures for on-chain actions.
{% endhint %}

***

## Recommended Mitigation

Include `msg.sender` (the expected creator) in the signed message so that the signature can only be used by the intended creator. Example change:

{% code title="SignatureVerifier.sol - include expectedCreator in verification" %}

```solidity
function checkAccessTokenInfo(
    address signer,
    address expectedCreator,  // Add this parameter
    AccessTokenInfo memory accessTokenInfo
) external view {
    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(
                    expectedCreator,  // Include msg.sender in hash
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    block.chainid
                )
            ),
            accessTokenInfo.signature
        ),
        InvalidSignature()
    );
}
```

{% endcode %}

Then update the call site in `Factory.produce()`:

{% code title="Factory.sol - pass msg.sender to signature check" %}

```solidity
factoryParameters.signerAddress.checkAccessTokenInfo(
    msg.sender,  // Pass the expected creator
    accessTokenInfo
);
```

{% endcode %}

This binds the signature to the intended creator address and prevents an attacker from reusing the signature from another sender.

***

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Legitimate creator requests signature

Alice wants to create an NFT collection "CoolNFTs" (COOL). She requests a signature from the backend signer with her collection parameters:

* name: "CoolNFTs"
* symbol: "COOL"
* contractURI: "ipfs\://..."
* feeNumerator: 500 (5% royalty)
  {% endstep %}

{% step %}

### Step 2 — Backend signs the parameters

The backend signs keccak256(name, symbol, contractURI, feeNumerator, chainid) and returns the signature to Alice.
{% endstep %}

{% step %}

### Step 3 — Alice submits transaction

Alice calls `Factory.produce()` with the signed `AccessTokenInfo`.
{% endstep %}

{% step %}

### Step 4 — Attacker monitors mempool

Bob (attacker) sees Alice's pending transaction and extracts:

* The signed `AccessTokenInfo` struct
* The signature
* All parameters
  {% endstep %}

{% step %}

### Step 5 — Attacker front-runs

Bob submits an identical transaction with higher gas price, calling `Factory.produce()` with the same `AccessTokenInfo` and signature.
{% endstep %}

{% step %}

### Step 6 — Attacker's transaction executes first

Because the signature does not validate `msg.sender`, the signature check passes and the collection is deployed with Bob as creator:

```solidity
// In Bob's transaction
AccessToken(nftAddress).initialize(
    /* ... */,
    creator: Bob,  // Bob becomes the creator!
    /* ... */
);
```

{% endstep %}

{% step %}

### Step 7 — Alice's transaction fails

When Alice's transaction eventually executes, it reverts with `TokenAlreadyExists()` because the same name/symbol (salt) was already used.
{% endstep %}

{% step %}

### Step 8 — Attacker profits

Bob now owns the "CoolNFTs" collection. When users mint NFTs, funds intended for Alice go to Bob:

```solidity
// In AccessToken._pay()
uint256 amountToCreator;
unchecked {
    amountToCreator = amount - fees;
}
// Funds go to Bob instead of Alice
_parameters.creator.safeTransferETH(amountToCreator);
```

{% endstep %}
{% endstepper %}

***

## References

* Factory.sol:produce() function
* SignatureVerifier.sol:checkAccessTokenInfo()
* AccessToken.sol:initialize()

***

If desired, I can produce a suggested patch diff for the repository to implement the mitigation (including updated function signatures and all call sites).
