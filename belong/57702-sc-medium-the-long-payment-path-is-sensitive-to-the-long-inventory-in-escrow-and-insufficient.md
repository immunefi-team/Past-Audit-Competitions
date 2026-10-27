# 57702 sc medium the long payment path is sensitive to the long inventory in escrow and insufficient inventory can easily lead to business unavailability dos of long payments&#x20;

* Submitted on Oct 28th 2025 at 10:03:44 UTC by @s8olidity for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* Report ID: #57702
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* Impacts: Smart contract unable to operate due to lack of token funds

## Description

### Brief / Intro

The payToVenue LONG payment branch first withdraws LONG from Escrow (distributeLONGDiscount), a "platform subsidy", then collects LONG from customers, and then proceeds with pledging, currency conversion, or transfers according to the rules. However, the LONG in Escrow comes solely from the `convenienceFeeLong` received from the venue's top-up, with no other stable replenishment. Even with a high volume of actual business, a shortfall in LONGs could occur, causing `payToVenue` to revert during the `distributeLONGDiscount` phase, resulting in a business interruption due to insufficient funds causing the contract to not function properly.

## Vulnerability Details

* Source Analysis:
  * `convenienceFeeAmount` (fixed at $5) is converted to a LONG value and injected into `Escrow.longDeposits` only within `venueDeposit`.
  * In the `payToVenue` LONG branch, `subsidyMinusFees` is first calculated as `platformSubsidyPercentage(amount) - processingFeePercentage(amount)` and then withdrawn via `escrow.distributeLONGDiscount(venue, this, subsidyMinusFees)`.
  * If `subsidyMinusFees > longDeposits[venue]`, the call reverts (`NotEnoughLONGs`), preventing subsequent customer transfers and pledge/currency swaps.
  * This is a shortcoming caused by "business parameters + fund flow design", not a permission or front-running issue. It is more likely to be triggered during peak or large-value payments.

## Impact Details

* The contract cannot accept LONG payment paths as expected → business interruption.
* Scope definition matches: Medium (Smart contract unable to operate due to lack of token funds).
* Associated Risks: Frequent rollbacks block customer payments, also blocking downstream actions such as promotions and referral points redemption (complete rollback).

## References

* contracts/v2/platform/BelongCheckIn.sol:508 (distributeLONGDiscount withdraws subsidy LONG)
* contracts/v2/periphery/Escrow\.sol:105 (NotEnoughLONGs check)
* contracts/v2/platform/BelongCheckIn.sol:396 (convenienceFeeLong is the only path to inject LONG)

## Proof of Concept

{% code title="test/v2/platform/belong-long-liquidity.test.ts" %}

```ts
import { ethers } from 'hardhat';
import { expect } from 'chai';
import EthCrypto from 'eth-crypto';
import {
  deployBelongCheckIn,
  deployEscrow,
  deployFactory,
  deployLONG,
  deployStaking,
  deployRoyaltiesReceiverV2Implementation,
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';
import { deploySignatureVerifier, deployHelper } from '../../../helpers/deployLibraries';
import { deployMockTransferValidatorV2, deployWETHMock } from '../../../helpers/deployMockFixtures';
import { BelongCheckIn, Factory, Escrow, LONG, Staking, CreditToken } from '../../../typechain-types';
import { BigNumber } from 'ethers';

describe('BelongCheckIn: LONG path can DoS on low Escrow LONG', () => {
  it('payToVenue (LONG) reverts when Escrow.longDeposits is insufficient', async () => {
    const [owner, venue, customer] = await ethers.getSigners();

    const signatureVerifier = await deploySignatureVerifier();
    const helper = await deployHelper();

    // Deploy BelongCheckIn with dummy payments config
    const paymentsInfo = {
      slippageBps: BigNumber.from(0), // irrelevant for revert point
      swapPoolFees: 3000,
      swapV3Factory: owner.address,
      swapV3Router: owner.address,
      swapV3Quoter: owner.address,
      wNativeCurrency: owner.address,
      usdc: (await deployWETHMock()).address, // mock
      long: (await deployLONG(owner.address, owner.address, owner.address)).address,
      maxPriceFeedDelay: BigNumber.from(3600),
    };

    const belong: BelongCheckIn = await deployBelongCheckIn(
      signatureVerifier.address,
      helper.address,
      owner.address,
      paymentsInfo as any
    );
    const escrow: Escrow = await deployEscrow(belong.address);

    // Wire contracts: factory / staking / credit tokens (simplified just to hit the branch)
    const rrImpl = await deployRoyaltiesReceiverV2Implementation();
    const accessImpl = await deployAccessTokenImplementation(signatureVerifier.address);
    const creditImpl = await deployCreditTokenImplementation();
    const vestingImpl = await deployVestingWalletImplementation();
    const validator = await deployMockTransferValidatorV2();

    const factory: Factory = await deployFactory(
      owner.address,
      owner.address, // signer not used here
      signatureVerifier.address,
      validator.address,
      {
        accessToken: accessImpl.address,
        creditToken: creditImpl.address,
        royaltiesReceiver: rrImpl.address,
        vestingWallet: vestingImpl.address,
      }
    );

    const longToken: LONG = await ethers.getContractAt('LONG', paymentsInfo.long);
    const staking: Staking = await deployStaking(owner.address, owner.address, longToken.address);

    // Set contracts
    await belong.setContracts({
      factory: factory.address,
      escrow: escrow.address,
      staking: staking.address,
      venueToken: (await ethers.getContractAt('CreditToken', (await (await factory.getCreditTokenInstanceInfo('','')).creditToken).toString())).address, // not used
      promoterToken: (await ethers.getContractAt('CreditToken', (await (await factory.getCreditTokenInstanceInfo('','')).creditToken).toString())).address, // not used
      longPF: owner.address,
    } as any);

    const fees = {
      referralCreditsAmount: 3,
      affiliatePercentage: 0,
      longCustomerDiscountPercentage: 0,
      platformSubsidyPercentage: 300, // 3%
      processingFeePercentage: 100,   // 1%
      buybackBurnPercentage: 0,
    };
    await belong.setParameters(paymentsInfo as any, fees as any, [
      { promoterStakingInfo: { usdcPercentage: 0, longPercentage: 0 }, venueStakingInfo: { depositFeePercentage: 0, convenienceFeeAmount: 0 } },
      { promoterStakingInfo: { usdcPercentage: 0, longPercentage: 0 }, venueStakingInfo: { depositFeePercentage: 0, convenienceFeeAmount: 0 } },
      { promoterStakingInfo: { usdcPercentage: 0, longPercentage: 0 }, venueStakingInfo: { depositFeePercentage: 0, convenienceFeeAmount: 0 } },
      { promoterStakingInfo: { usdcPercentage: 0, longPercentage: 0 }, venueStakingInfo: { depositFeePercentage: 0, convenienceFeeAmount: 0 } },
      { promoterStakingInfo: { usdcPercentage: 0, longPercentage: 0 }, venueStakingInfo: { depositFeePercentage: 0, convenienceFeeAmount: 0 } },
    ] as any);


    const customerInfo = {
      paymentInUSDC: false,
      visitBountyAmount: 0,
      spendBountyPercentage: 0,
      customer: customer.address,
      venueToPayFor: venue.address,
      promoter: ethers.constants.AddressZero,
      amount: ethers.utils.parseEther('100'), 
      signature: '0x',
    };

    await longToken.connect(customer).approve(belong.address, customerInfo.amount);

    await expect(belong.payToVenue(customerInfo as any)).to.be.reverted;
  });
});
```

{% endcode %}
