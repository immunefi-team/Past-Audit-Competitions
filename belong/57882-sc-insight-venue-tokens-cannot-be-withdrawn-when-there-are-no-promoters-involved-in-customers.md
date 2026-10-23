# 57882 sc insight venue tokens cannot be withdrawn when there are no promoters involved in customers transactions

**Submitted on Oct 29th 2025 at 11:14:58 UTC by @Josh4324 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57882
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

In the BelongCheckIn contract, venue owners deposit USDC and LONG tokens into an Escrow, receiving ERC1155 venue tokens (via CreditToken) representing their USD-denominated balances. These venue tokens are primarily burned to mint promoter tokens during customer payments involving promoters, with promoters later claiming payouts from Escrow. However, if no promoters are involved (i.e., no customer payments trigger burns), venue owners have no mechanism to burn their venue tokens and redeem the underlying funds from Escrow.

This results in permanent locking of deposited funds, as there are no direct redemption or burn-to-withdraw functions for venue token.

## Vulnerability Details

Venue deposits mint venue tokens proportional to USD value, but burns only occur in `payToVenue` when a promoter is specified (burning venue tokens to mint promoter tokens). Without promoters, no burns happen, and there is no alternative function for venues to initiate burns or directly withdraw from Escrow. The Escrow only disburses via `distributeVenueDeposit` (for promoter claims) or `distributeLONGDiscount` (subsidies), both tied to other actions. The emergency `emergencyCancelPayment` only reverses promoter credits back to venue tokens, not redeeming underlying funds.

### Scenario for Issue

{% stepper %}
{% step %}

### Venue deposits

* Venue deposits USDC/LONG into Escrow.
* Venue receives ERC1155 venue tokens representing the USD-denominated balance.
  {% endstep %}

{% step %}

### No promoter involvement

* No customer payments involve promoters (e.g., direct payments or no promoter ecosystem adoption).
* Customer payments do not specify a promoter, so `payToVenue` does not burn venue tokens to mint promoter tokens.
  {% endstep %}

{% step %}

### Funds become stuck

* Venue tokens remain unburned.
* There is no function allowing venue owners to burn venue tokens to redeem underlying funds from Escrow.
* Deposited funds remain locked indefinitely.
  {% endstep %}
  {% endstepper %}

### Conditions for Issue

* No promoter involvement in customer payments.
* Venue wishes to exit/redeem deposits.

## Impact Details

{% hint style="danger" %}
Venue owners can face permanent funds lockup if no promoters are involved in customer transactions, being unable to recover deposits held in Escrow.
{% endhint %}

## References

* Vulnerable code location: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol#L379>

## Proof of Concept

Copy the code below to `file.test.ts` in `test/v2/platform` and run:

yarn test test/v2/platform/file.test.ts

```ts
import { ethers } from 'hardhat';
import { BigNumber } from 'ethers';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import EthCrypto from 'eth-crypto';
import {
  AccessToken,
  CreditToken,
  Escrow,
  Factory,
  Helper,
  MockTransferValidatorV2,
  RoyaltiesReceiverV2,
  SignatureVerifier,
  Staking,
  BelongCheckIn,
  VestingWalletExtended,
} from '../../../typechain-types';
import {
  deployCreditTokens,
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployFactory,
  deployRoyaltiesReceiverV2Implementation,
  deployStaking,
  deployBelongCheckIn,
  deployEscrow,
  deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';
import { getSignerFromAddress, getToken, startSimulateMainnet, stopSimulate } from '../../../helpers/fork';
import { deployHelper, deploySignatureVerifier } from '../../../helpers/deployLibraries';
import { deployMockTransferValidatorV2, deployPriceFeeds } from '../../../helpers/deployMockFixtures';
import { expect } from 'chai';
import {
  CustomerInfoStruct,
  PromoterInfoStruct,
  VenueInfoStruct,
  VenueRulesStruct,
} from '../../../typechain-types/contracts/v2/platform/BelongCheckIn';
import { U, u } from '../../../helpers/math';

describe('BelongCheckIn ETH Uniswap', () => {
  const chainId = 31337;

  const WETH_ADDRESS = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2';
  const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';
  const ENA_ADDRESS = '0x57e114B691Db790C35207b2e685D4A43181e6061'; //used instead of LONG

  const USDC_WHALE_ADDRESS = '0x8EB8a3b98659Cce290402893d0123abb75E3ab28';
  const WETH_WHALE_ADDRESS = '0x57757E3D981446D585Af0D9Ae4d7DF6D64647806';
  const ENA_WHALE_ADDRESS = '0xF977814e90dA44bFA03b6295A0616a897441aceC';

  const UNISWAP_FACTORY_ADDRESS = '0x1F98431c8aD98523631AE4a59f267346ea31F984';
  const UNISWAP_ROUTER_ADDRESS = '0xE592427A0AEce92De3Edee1F18E0157C05861564';
  const UNISWAP_QUOTER_ADDRESS = '0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6';
  const POOL_FEE = 3000;
  const MAX_PRICEFEED_DELAY = 3600;

  const usdcPercentage = 2000;
  const convenienceFeeAmount = U(5, 6); // $5

  const paymentsInfo: BelongCheckIn.PaymentsInfoStruct = {
    swapPoolFees: POOL_FEE,
    slippageBps: BigNumber.from(10).pow(27).sub(1),
    swapV3Factory: UNISWAP_FACTORY_ADDRESS,
    swapV3Router: UNISWAP_ROUTER_ADDRESS,
    swapV3Quoter: UNISWAP_QUOTER_ADDRESS,
    wNativeCurrency: WETH_ADDRESS,
    usdc: USDC_ADDRESS,
    long: ENA_ADDRESS,
    maxPriceFeedDelay: MAX_PRICEFEED_DELAY,
  };

  const stakingRewards: [
    BelongCheckIn.RewardsInfoStruct,
    BelongCheckIn.RewardsInfoStruct,
    BelongCheckIn.RewardsInfoStruct,
    BelongCheckIn.RewardsInfoStruct,
    BelongCheckIn.RewardsInfoStruct,
  ] = [
    {
      venueStakingInfo: {
        depositFeePercentage: 1000,
        convenienceFeeAmount,
      } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: {
        usdcPercentage,
        longPercentage: 800,
      } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: {
        depositFeePercentage: 900,
        convenienceFeeAmount,
      } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: {
        usdcPercentage,
        longPercentage: 700,
      } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: {
        depositFeePercentage: 800,
        convenienceFeeAmount,
      } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: {
        usdcPercentage,
        longPercentage: 600,
      } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: {
        depositFeePercentage: 700,
        convenienceFeeAmount,
      } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: {
        usdcPercentage,
        longPercentage: 500,
      } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: {
        depositFeePercentage: 500,
        convenienceFeeAmount,
      } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: {
        usdcPercentage,
        longPercentage: 400,
      } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
  ];
  const fees: BelongCheckIn.FeesStruct = {
    referralCreditsAmount: 3,
    affiliatePercentage: 1000,
    longCustomerDiscountPercentage: 300,
    platformSubsidyPercentage: 300,
    processingFeePercentage: 250,
    buybackBurnPercentage: 5000,
  };
  let implementations: Factory.ImplementationsStruct, contracts: BelongCheckIn.ContractsStruct;

  before(startSimulateMainnet);
  after(stopSimulate);

  async function fixture() {
    const [admin, treasury, manager, minter, burner, pauser, referral] = await ethers.getSigners();
    const signer = EthCrypto.createIdentity();

    const WETH_whale = await getSignerFromAddress(WETH_WHALE_ADDRESS);
    const USDC_whale = await getSignerFromAddress(USDC_WHALE_ADDRESS);
    const ENA_whale = await getSignerFromAddress(ENA_WHALE_ADDRESS);
    const WETH = await getToken(WETH_ADDRESS);
    const USDC = await getToken(USDC_ADDRESS);
    const ENA = await getToken(ENA_ADDRESS);

    const signatureVerifier: SignatureVerifier = await deploySignatureVerifier();
    const validator: MockTransferValidatorV2 = await deployMockTransferValidatorV2();
    const accessTokenImplementation: AccessToken = await deployAccessTokenImplementation(signatureVerifier.address);
    const royaltiesReceiverV2Implementation: RoyaltiesReceiverV2 = await deployRoyaltiesReceiverV2Implementation();
    const creditTokenImplementation: CreditToken = await deployCreditTokenImplementation();
    const vestingWallet: VestingWalletExtended = await deployVestingWalletImplementation();

    const treasuryUsdcBalance = await USDC.balanceOf(treasury.address);
    if (!treasuryUsdcBalance.isZero()) {
      await USDC.connect(treasury).transfer(USDC_whale.address, treasuryUsdcBalance);
    }

    implementations = {
      accessToken: accessTokenImplementation.address,
      creditToken: creditTokenImplementation.address,
      royaltiesReceiver: royaltiesReceiverV2Implementation.address,
      vestingWallet: vestingWallet.address,
    };

    const factory: Factory = await deployFactory(
      treasury.address,
      signer.address,
      signatureVerifier.address,
      validator.address,
      implementations,
    );

    const helper: Helper = await deployHelper();
    const staking: Staking = await deployStaking(admin.address, treasury.address, ENA_ADDRESS);

    const referralCode = EthCrypto.hash.keccak256([
      { type: 'address', value: referral.address },
      { type: 'address', value: factory.address },
      { type: 'uint256', value: chainId },
    ]);

    await factory.connect(referral).createReferralCode();

    const belongCheckIn: BelongCheckIn = await deployBelongCheckIn(
      signatureVerifier.address,
      helper.address,
      admin.address,
      paymentsInfo,
    );

    const escrow: Escrow = await deployEscrow(belongCheckIn.address);
    const { pf1, pf2, pf2_2, pf2_3, pf3 } = await deployPriceFeeds();
    const { venueToken, promoterToken } = await deployCreditTokens(
      true,
      false,
      factory.address,
      signer.privateKey,
      admin,
      manager.address,
      belongCheckIn.address,
      belongCheckIn.address,
    );

    contracts = {
      factory: factory.address,
      escrow: escrow.address,
      staking: staking.address,
      venueToken: venueToken.address,
      promoterToken: promoterToken.address,
      longPF: pf1.address,
    };

    await belongCheckIn.setContracts(contracts);

    return {
      signatureVerifier,
      helper,
      factory,
      staking,
      venueToken,
      promoterToken,
      belongCheckIn,
      escrow,
      pf1,
      pf2,
      pf2_2,
      pf2_3,
      pf3,
      admin,
      treasury,
      manager,
      minter,
      burner,
      pauser,
      referral,
      signer,
      referralCode,
      WETH,
      USDC,
      ENA,
      WETH_whale,
      USDC_whale,
      ENA_whale,
    };
  }

  describe('Funds Stuck', () => {
    it('Funds Stuck', async () => {
      const {
        belongCheckIn,
        helper,
        escrow,
        promoterToken,
        signatureVerifier,
        referral,
        signer,
        treasury,
        referralCode,
        USDC,
        USDC_whale,
        ENA_whale,
      } = await loadFixture(fixture);

      await belongCheckIn.setParameters(paymentsInfo, fees, stakingRewards);

      const uri = 'uriuri';
      const venueAmount = await u(100, USDC);
      const venue = USDC_whale.address;
      const venueMessage = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, referralCode, uri, chainId],
      );
      const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);

      let venueInfo: VenueInfoStruct = {
        rules: { paymentType: 3, bountyType: 3, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode,
        uri,
        signature: venueSignature,
      };

      const venueBalance_after = await USDC.balanceOf(escrow.address);

      console.log('Before Venue Payment', venueBalance_after);

      const paymentToAffiliate = await helper.calculateRate(fees.affiliatePercentage, venueAmount);
      const willBeTaken = paymentToAffiliate.add(convenienceFeeAmount.add(venueAmount));
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      const customerAmount = await u(100, USDC);
      const customerMessage = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [
          true, // paymentInUSDC
          await u(1, USDC), // visitBountyAmount (uint24, adjust to uint256 if needed)
          1000, // spendBountyPercentage (uint24, adjust to uint256 if needed)
          ENA_whale.address, // customer
          USDC_whale.address, // venueToPayFor
          ethers.constants.AddressZero, // promoter
          customerAmount, // amount
          chainId, // block.chainid
        ],
      );
      const customerSignature = EthCrypto.sign(signer.privateKey, customerMessage);
      const customerInfo: CustomerInfoStruct = {
        paymentInUSDC: true,
        visitBountyAmount: await u(1, USDC),
        spendBountyPercentage: 1000,
        customer: ENA_whale.address,
        venueToPayFor: USDC_whale.address,
        promoter: ethers.constants.AddressZero,
        amount: customerAmount,
        signature: customerSignature,
      };

      await USDC.connect(USDC_whale).transfer(ENA_whale.address, customerAmount);
      await USDC.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);

      await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);

      const venueBalance_after1 = await USDC.balanceOf(escrow.address);

      console.log('After Venue Payment', venueBalance_after1);
    });
  });
});
```

Observed output when running test:

Before Venue Payment BigNumber { value: "0" } After Venue Payment BigNumber { value: "100000000" } ✔ Funds Stuck (1514ms)
