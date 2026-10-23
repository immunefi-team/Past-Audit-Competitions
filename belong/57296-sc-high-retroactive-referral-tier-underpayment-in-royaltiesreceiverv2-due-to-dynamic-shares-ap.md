# 57296 sc high retroactive referral tier underpayment in royaltiesreceiverv2 due to dynamic shares applied to historical funds

**Submitted on Oct 25th 2025 at 03:18:25 UTC by @Rhaydden for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57296
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/RoyaltiesReceiverV2.sol>
* **Impacts:**
  * Theft of unclaimed royalties

## Description

### Issue description

`RoyaltiesReceiverV2` computes each payee’s owed amount using the current shares at the time of release, not the shares that applied when funds arrived. Referral shares depend on the creator’s current tier which is derived from `usedCode[creator][code]`. When the creator uses the same referral code again (deploying another collection), the tier changes and `shares()` yields a new split. `_pendingPayment` retroactively applies this new split to all historical funds causing misallocation.

This dynamic shares logic (tied to current factory parameters and current referral tier) is implemented here:

<https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/RoyaltiesReceiverV2.sol#L107-L132>

```solidity
function shares(address account) public view returns (uint256) {
        RoyaltiesReceivers memory _royaltiesReceivers = royaltiesReceivers;

        Factory _factory = factory;
        Factory.RoyaltiesParameters memory royaltiesParameters = _factory.royaltiesParameters();
        if (account == _royaltiesReceivers.creator) {
            return royaltiesParameters.amountToCreator;
        } else {
            uint256 platformShare = royaltiesParameters.amountToPlatform;
            uint256 referralShare;
            if (_royaltiesReceivers.referral != address(0)) {
                referralShare = _factory.getReferralRate(
                    _royaltiesReceivers.creator, referralCode, royaltiesParameters.amountToPlatform
                );

                if (referralShare > 0) {
                    unchecked {
                        platformShare -= referralShare;
                    }
                }
            }
            return account == _royaltiesReceivers.platform
                ? platformShare
                : account == _royaltiesReceivers.referral ? referralShare : 0;
        }
    }
```

The retroactive PaymentSplitter math applies the current share to total historical receipts here:

<https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/RoyaltiesReceiverV2.sol#L227-L238>

```solidity
   function _pendingPayment(bool isNativeRelease, address token, address account) private view returns (uint256) {
        Releases storage releases = isNativeRelease ? nativeReleases : erc20Releases[token];
        uint256 balance = isNativeRelease ? address(this).balance : token.balanceOf(address(this));

        uint256 payment = ((balance + releases.totalReleased) * shares(account)) / TOTAL_SHARES;

        if (payment <= releases.released[account]) {
            return 0;
        }

        return payment - releases.released[account];
    }
```

Tier changes when the creator uses the same referral code again here:

<https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/extensions/ReferralSystemV2.sol#L157-L165>

```solidity
function _setReferralUser(bytes32 hashedCode, address referralUser) internal {
    ...
    uint8 used = usedCode[referralUser][hashedCode];
    if (used < MAX_TIER_INDEX) {
        unchecked { usedCode[referralUser][hashedCode] = used + 1; }
    }
    emit ReferralCodeUsed(hashedCode, referralUser);
}
```

The factory increments usage during collection creation in `factory.sol`:

<https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/Factory.sol#L254>

```solidity
_setReferralUser(referralCode, msg.sender);
```

## Impact

High - Theft of unclaimed royalties

When the referral tier decreases later, the contract underpays the referral and overpays the platform on historical funds that were already in the receiver but unclaimed. The creator’s share remains correct. The misallocation occurs between referral and platform.

Example with default config (creator 80%, platform 20%; referral tiers: 50% -> 30% of platform share):

* Two equal inflows: 1000 ETH + 1000 ETH
* Correct per-period:
  * Referral: 100 + 60 = 160
  * Platform: 100 + 140 = 240
  * Creator: 800 + 800 = 1600
* Actual with retroactive dynamic shares (after tier drops to 30% before release):
  * Referral: 2000 × 6% = 120 (underpaid by 40)
  * Platform: 2000 × 14% = 280 (overpaid by 40)
  * Creator: 2000 × 80% = 1600

## Recommended mitigation steps

Snapshot shares per receiver at initialization. Store creator/platform/referral shares once during `initialize()` (based on then-current tier and royalties parameters) and make `shares()` return those stored constants. This follows v1 behavior and removes retroactive drift.

## Proof of Concept

Paste the PoC below in `test/v2/periphery/royalties-receiver-v2.poc.test.ts`:

```ts
import { ethers } from 'hardhat';
import { expect } from 'chai';
import EthCrypto from 'eth-crypto';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';

import {
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployFactory,
  deployRoyaltiesReceiverV2Implementation,
  deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';
import { deployMockTransferValidatorV2 } from '../../../helpers/deployMockFixtures';
import { deploySignatureVerifier } from '../../../helpers/deployLibraries';
import { AccessToken, Factory, RoyaltiesReceiverV2 } from '../../../typechain-types';
import { hashAccessTokenInfo } from '../../../helpers/math';

// This PoC demonstrates retroactive underpayment to referral when the referral tier decreases
// due to additional uses of the same referral code by the creator. The RoyaltiesReceiverV2
// computes pending payments with the CURRENT dynamic shares over total historical receipts.
// With default tiers [0, 5000, 3000, 1500, 500] and platform=20%, referral is underpaid.

describe('PoC - RoyaltiesReceiverV2 retroactive referral underpayment', function () {
  const NATIVE_CURRENCY_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';

  async function fixture() {
    const [owner, creator, referrer] = await ethers.getSigners();

    // Off-chain signer used by the Factory to authorize collection creation
    const signer = EthCrypto.createIdentity();

    // Deploy libs and implementations
    const signatureVerifier = await deploySignatureVerifier();
    const validator = await deployMockTransferValidatorV2();
    const accessTokenImpl = await deployAccessTokenImplementation(signatureVerifier.address);
    const rrImpl = await deployRoyaltiesReceiverV2Implementation();
    const creditTokenImpl = await deployCreditTokenImplementation();
    const vestingWalletImpl = await deployVestingWalletImplementation();

    const implementations: Factory.ImplementationsStruct = {
      accessToken: accessTokenImpl.address,
      creditToken: creditTokenImpl.address,
      royaltiesReceiver: rrImpl.address,
      vestingWallet: vestingWalletImpl.address,
    };

    const factory: Factory = await deployFactory(
      owner.address,
      signer.address,
      signatureVerifier.address,
      validator.address,
      implementations,
    );

    // Create referral code for referrer
    await (await factory.connect(referrer).createReferralCode()).wait();
    const referralCode = await factory.getReferralCodeByCreator(referrer.address);

    return { owner, creator, referrer, signer, factory, referralCode };
  }

  it('underpays referral when tier decreases between inflows', async () => {
    const { owner, creator, referrer, signer, factory, referralCode } = await loadFixture(fixture);

    // Deploy first collection for creator using referral code (used count becomes 1 → 50%)
    const chainId = (await ethers.provider.getNetwork()).chainId;

    // Prepare AccessToken creation data and signature (use helpers to match on-chain verifier)
    const nameA = 'CollectionA';
    const symbolA = 'COLA';
    const contractURIA = 'uri/collectionA';
    const feeNumerator = 600; // matches helper default
    const messageA = hashAccessTokenInfo(nameA, symbolA, contractURIA, feeNumerator, chainId);
    const sigA = EthCrypto.sign(signer.privateKey, messageA);

    const accessTokenParamsA = {
      metadata: { name: nameA, symbol: symbolA },
      contractURI: contractURIA,
      paymentToken: NATIVE_CURRENCY_ADDRESS,
      mintPrice: ethers.utils.parseEther('0.01'),
      whitelistMintPrice: ethers.utils.parseEther('0.01'),
      transferable: true,
      maxTotalSupply: ethers.BigNumber.from('1000'),
      feeNumerator,
      collectionExpire: ethers.BigNumber.from('86400'),
      signature: sigA,
    };

    await (await factory.connect(creator).produce(accessTokenParamsA, referralCode)).wait();
    const instanceInfoA = await factory.nftInstanceInfo(nameA, symbolA);
    const royaltiesReceiverA: RoyaltiesReceiverV2 = await ethers.getContractAt(
      'RoyaltiesReceiverV2',
      instanceInfoA.royaltiesReceiver,
    );

    const payees = await royaltiesReceiverA.royaltiesReceivers();
    const creatorShareInitial = await royaltiesReceiverA.shares(payees.creator);
    const platformShareInitial = await royaltiesReceiverA.shares(payees.platform);
    const referralShareInitial =
      payees.referral === ethers.constants.AddressZero ? ethers.constants.Zero : await royaltiesReceiverA.shares(payees.referral);

    // Sanity: with default config, initial shares should be 8000/1000/1000 (creator/platform/referral)
    expect(creatorShareInitial).to.eq(8000);
    expect(platformShareInitial).to.eq(1000);
    expect(referralShareInitial).to.eq(1000);

    // Inflow #1: send 1000 ETH to receiver A
    await (await owner.sendTransaction({ to: royaltiesReceiverA.address, value: ethers.utils.parseEther('1000') })).wait();

    // Bump the referral tier by producing another collection with the same code
    const nameB = 'CollectionB';
    const symbolB = 'COLB';
    const contractURIB = 'uri/collectionB';
    const messageB = hashAccessTokenInfo(nameB, symbolB, contractURIB, feeNumerator, chainId);
    const sigB = EthCrypto.sign(signer.privateKey, messageB);

    const accessTokenParamsB = {
      metadata: { name: nameB, symbol: symbolB },
      contractURI: contractURIB,
      paymentToken: NATIVE_CURRENCY_ADDRESS,
      mintPrice: ethers.utils.parseEther('0.02'),
      whitelistMintPrice: ethers.utils.parseEther('0.02'),
      transferable: true,
      maxTotalSupply: ethers.BigNumber.from('1000'),
      feeNumerator,
      collectionExpire: ethers.BigNumber.from('86400'),
      signature: sigB,
    };

    // This increments usedCode[creator][referralCode] from 1 to 2, lowering referral share to 600
    await (await factory.connect(creator).produce(accessTokenParamsB, referralCode)).wait();

    // Inflow #2: send another 1000 ETH to receiver A
    await (await owner.sendTransaction({ to: royaltiesReceiverA.address, value: ethers.utils.parseEther('1000') })).wait();

    // Assert the current dynamic shares reflect tier 2 (platform 1400, referral 600)
    const platformShareAfter = await royaltiesReceiverA.shares(payees.platform);
    const referralShareAfter = await royaltiesReceiverA.shares(payees.referral);
    expect(platformShareAfter).to.eq(1400);
    expect(referralShareAfter).to.eq(600);

    // Release all native funds to payees
    await (await royaltiesReceiverA.connect(owner).releaseAll(NATIVE_CURRENCY_ADDRESS)).wait();

    // Read released amounts from the contract (gas-neutral way to assert)
    const releasedCreator = await royaltiesReceiverA.released(NATIVE_CURRENCY_ADDRESS, payees.creator);
    const releasedPlatform = await royaltiesReceiverA.released(NATIVE_CURRENCY_ADDRESS, payees.platform);
    const releasedReferral = await royaltiesReceiverA.released(NATIVE_CURRENCY_ADDRESS, payees.referral);
    const releasedTotal = await royaltiesReceiverA.totalReleased(NATIVE_CURRENCY_ADDRESS);

    // Actual distribution under dynamic/retroactive shares (tier 2 applied to both inflows):
    // creator: 2000 * 80% = 1600
    // platform: 2000 * 14% = 280
    // referral: 2000 * 6% = 120
    expect(releasedCreator).to.eq(ethers.utils.parseEther('1600'));
    expect(releasedPlatform).to.eq(ethers.utils.parseEther('280'));
    expect(releasedReferral).to.eq(ethers.utils.parseEther('120'));
    expect(releasedTotal).to.eq(ethers.utils.parseEther('2000'));

    // Correct per-period distribution (non-retroactive):
    // inflow1 (tier1): creator=800, platform=100, referral=100
    // inflow2 (tier2): creator=800, platform=140, referral=60
    // expected totals: creator=1600, platform=240, referral=160
    const expectedCreator = ethers.utils.parseEther('1600');
    const expectedPlatform = ethers.utils.parseEther('240');
    const expectedReferral = ethers.utils.parseEther('160');

    // Show the delta that represents the bug impact
    expect(releasedCreator).to.eq(expectedCreator);
    expect(releasedPlatform).to.eq(expectedPlatform.add(ethers.utils.parseEther('40'))); // overpaid by 40
    expect(releasedReferral).to.eq(expectedReferral.sub(ethers.utils.parseEther('40'))); // underpaid by 40
  });
});
```

Run with:

```bash
pnpm install
NO_FORK=true PK=0x1111111111111111111111111111111111111111111111111111111111111111 LEDGER_ADDRESS=0x0000000000000000000000000000000000000001 pnpm hardhat test test/v2/periphery/royalties-receiver-v2.poc.test.ts
```

### Logs

```
PoC - RoyaltiesReceiverV2 retroactive referral underpayment
  ✔ underpays referral when tier decreases between inflows (540ms)

1 passing
```

The test asserts:

* creator: 1600 ETH
* platform: 280 ETH
* referral: 120 ETH
* total: 2000 ETH
* Deltas vs correct: platform +40, referral −40
