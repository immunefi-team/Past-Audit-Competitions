# 57775 sc medium paytovenue will revert due to notenoughlongs funds in the escrow contract

**Submitted on Oct 28th 2025 at 20:30:45 UTC by @Josh4324 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57775
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief / Intro

The `payToVenue` function in the `BelongCheckIn` contract processes customer payments to venues, supporting both USDC and LONG tokens while applying platform subsidies, discounts, and promoter rewards. When a customer pays with LONG, the function attempts to distribute a platform subsidy from the venue's escrow balance. If the escrow lacks sufficient LONG tokens to cover the subsidy, the call reverts with a `NotEnoughLONGs` error. This creates a dependency on external escrow funding and can cause valid customer payments to fail. The root cause is missing pre-checks or fallback handling for insufficient escrow balances.

### Vulnerability Details

Relevant excerpt from `payToVenue`:

```sol
function payToVenue(CustomerInfo calldata customerInfo) external {
        BelongCheckInStorage memory _storage = belongCheckInStorage;
        VenueRules memory rules = generalVenueInfo[customerInfo.venueToPayFor].rules;

        _storage.contracts.factory.nftFactoryParameters().signerAddress.checkCustomerInfo(customerInfo, rules);

        uint256 venueId = customerInfo.venueToPayFor.getVenueId();

        if (customerInfo.promoter != address(0)) {
            uint256 rewardsToPromoter = customerInfo.paymentInUSDC
                ? customerInfo.visitBountyAmount + customerInfo.spendBountyPercentage.calculateRate(customerInfo.amount)
                : _storage.paymentsInfo.usdc
                    .unstandardize(
                        // standardization
                        _storage.paymentsInfo.usdc.standardize(customerInfo.visitBountyAmount)
                            + customerInfo.spendBountyPercentage
                                .calculateRate(
                                    _storage.paymentsInfo.long
                                        .getStandardizedPrice(
                                            _storage.contracts.longPF,
                                            customerInfo.amount,
                                            _storage.paymentsInfo.maxPriceFeedDelay
                                        )
                                )
                    );
            uint256 venueBalance = _storage.contracts.venueToken.balanceOf(customerInfo.venueToPayFor, venueId);
            require(venueBalance >= rewardsToPromoter, NotEnoughBalance(rewardsToPromoter, venueBalance));

            _storage.contracts.venueToken.burn(customerInfo.venueToPayFor, venueId, rewardsToPromoter);
            _storage.contracts.promoterToken
                .mint(customerInfo.promoter, venueId, rewardsToPromoter, _storage.contracts.venueToken.uri(venueId));
        }

        if (customerInfo.paymentInUSDC) {
            _storage.paymentsInfo.usdc
                .safeTransferFrom(customerInfo.customer, customerInfo.venueToPayFor, customerInfo.amount);
        } else {
            // platform subsidy - processing fee
            uint256 subsidyMinusFees =
                _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount)
                - _storage.fees.processingFeePercentage.calculateRate(customerInfo.amount);
 @>           _storage.contracts.escrow
                .distributeLONGDiscount(customerInfo.venueToPayFor, address(this), subsidyMinusFees);

            // customer paid amount - longCustomerDiscountPercentage (3%)
            uint256 longFromCustomer =
                customerInfo.amount - _storage.fees.longCustomerDiscountPercentage.calculateRate(customerInfo.amount);
            _storage.paymentsInfo.long.safeTransferFrom(customerInfo.customer, address(this), longFromCustomer);

            uint256 longAmount = subsidyMinusFees + longFromCustomer;

            if (rules.longPaymentType == LongPaymentTypes.AutoStake) {
                // Approve only what is needed, then clear allowance after deposit.
                _storage.paymentsInfo.long.safeApproveWithRetry(address(_storage.contracts.staking), longAmount);
                _storage.contracts.staking.deposit(longAmount, customerInfo.venueToPayFor);
                _storage.paymentsInfo.long.safeApprove(address(_storage.contracts.staking), 0);
            } else if (rules.longPaymentType == LongPaymentTypes.AutoConvert) {
                _swapLONGtoUSDC(customerInfo.venueToPayFor, longAmount);
            } else {
                _storage.paymentsInfo.long.safeTransfer(customerInfo.venueToPayFor, longAmount);
            }
        }

        emit CustomerPaid(
            customerInfo.customer,
            customerInfo.venueToPayFor,
            customerInfo.promoter,
            customerInfo.amount,
            customerInfo.visitBountyAmount,
            customerInfo.spendBountyPercentage
        );
    }
```

In the LONG payment branch the contract computes `subsidyMinusFees` and immediately calls:

```sol
_storage.contracts.escrow.distributeLONGDiscount(customerInfo.venueToPayFor, address(this), subsidyMinusFees);
```

`Escrow.distributeLONGDiscount` checks the venue's LONG deposits and reverts if the escrow has insufficient LONG balance:

```sol
require(longDeposits >= amount, NotEnoughLONGs(...));
```

Because `payToVenue` does not check the venue escrow balance before calling `distributeLONGDiscount` and has no fallback behavior, the entire `payToVenue` call reverts after partial execution when the escrow is underfunded.

### Impact

{% hint style="warning" %}
Customers paying in LONG can experience unexpected transaction reverts when the associated venue escrow lacks sufficient LONG tokens to cover the subsidy. This prevents valid customer payments from completing, interrupting normal operation of the contract.
{% endhint %}

### References

* Vulnerable file: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol#L435>

## Proof of Concept (test)

Place this test into `test/v2/platform/file.test.ts` and run `yarn test test/v2/platform/file.test.ts`

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

  const usdcPercentage = 1000;
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

  describe('Customer flow long payment', () => {
    it('payToVenue() (w/o promoter)', async () => {
      const { belongCheckIn, escrow, helper, signer, USDC, ENA, USDC_whale, ENA_whale } = await loadFixture(fixture);

      // Pay venue deposit

      const uri = 'uriuri';
      const venueAmount = await u(1, USDC);
      const venue = USDC_whale.address;
      const venueMessage = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 2, bountyType: 0, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature: venueSignature,
      };
      const willBeTaken = convenienceFeeAmount.add(venueAmount);
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      // Customer 1 Pays to venue

      const customerAmount = ethers.utils.parseEther('1000');

      const customerMessage = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [
          false, // paymentInUSDC
          0, // visitBountyAmount (uint24, adjust to uint256 if needed)
          0, // spendBountyPercentage (uint24, adjust to uint256 if needed)
          ENA_whale.address, // customer
          USDC_whale.address, // venueToPayFor
          ethers.constants.AddressZero, // promoter
          customerAmount, // amount
          chainId, // block.chainid
        ],
      );
      const customerSignature = EthCrypto.sign(signer.privateKey, customerMessage);
      const customerInfo: CustomerInfoStruct = {
        paymentInUSDC: false,
        visitBountyAmount: 0,
        spendBountyPercentage: 0,
        customer: ENA_whale.address,
        venueToPayFor: USDC_whale.address,
        promoter: ethers.constants.AddressZero,
        amount: customerAmount,
        signature: customerSignature,
      };

      await ENA.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);

      const tx = await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);

      await ENA.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);

      const tx1 = await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);
    });
  });
});
```

Observed failing result:

```
1) BelongCheckIn ETH Uniswap
       Customer flow long payment
         payToVenue() (w/o promoter):
     Error: VM Exception while processing transaction: reverted with custom error 'NotEnoughLONGs(2957086772067060989, 5000000000000000000)'
    at Escrow.distributeLONGDiscount (contracts/v2/periphery/Escrow.sol:115)
    at <UnrecognizedContract>.<unknown> (0x3f36ff84958b0473925afe368cd25a9c23fa56cf)
    at BelongCheckIn.payToVenue (contracts/v2/platform/BelongCheckIn.sol:476)
    at <UnrecognizedContract>.<unknown> (0x3c71ddff2325551660c7089e5e085589b62a4e7a)
    at EdrProviderWrapper.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:359:41)
    at async EthersProviderWrapper.send (node_modules/@nomiclabs/hardhat-ethers/src/internal/ethers-provider-wrapper.ts:13:20)
```

\-- End of report.
