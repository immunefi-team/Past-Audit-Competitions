# 57872 sc low processing fee computed on full long amount instead of subsidy in paytovenue underpaying venues and enabling long payment dos under misconfiguration

**Submitted on Oct 29th 2025 at 10:24:40 UTC by @Rhaydden for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57872
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>

## Description

### Issue description

In the LONG payment branch of `BelongCheckIn.payToVenue`, the `processingFeePercentage` is applied to the full pre-discount `customerInfo.amount` instead of to the platform subsidy.

This is not in line with the doc that says “processingFeePercentage: portion of LONG subsidy collected by the platform as processing fee.” Therefore, `processingFee` should be a percentage of `grossSubsidy`, not a percentage of the full `amount`.

Reference in code: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/BelongCheckIn.sol#L471-L485>

Relevant snippet (original behavior):

```solidity
// contracts/v2/platform/BelongCheckIn.sol
// ...
// platform subsidy - processing fee
uint256 subsidyMinusFees =
    _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount)
    - _storage.fees.processingFeePercentage.calculateRate(customerInfo.amount);
_storage.contracts.escrow
    .distributeLONGDiscount(customerInfo.venueToPayFor, address(this), subsidyMinusFees);

// customer paid amount - longCustomerDiscountPercentage (3%)
uint256 longFromCustomer =
    customerInfo.amount - _storage.fees.longCustomerDiscountPercentage.calculateRate(customerInfo.amount);
// ...
uint256 longAmount = subsidyMinusFees + longFromCustomer;
// ...
```

* With defaults (3% subsidy, 2.5% processing, 3% discount):
  * Buggy: `processingFee = 2.5% × full_LONG`
  * Correct: `processingFee = 2.5% × (3% × full_LONG)`

Result: the venue is underpaid. The missing LONG is not transferred to anyone and it remains in the venue’s escrowed `longDeposits`.\
If `processingFeePercentage > platformSubsidyPercentage`, the subtraction underflows and LONG payments revert, preventing redemption of escrow subsidy amounts.

{% hint style="warning" %}
Impact: Low — Contract fails to deliver promised returns, but doesn't lose value. Venues receive less LONG than intended. The difference remains in escrow. If misconfigured (processing fee larger than subsidy), LONG payments revert.
{% endhint %}

## Recommended mitigation steps

Apply processing fee to the subsidy, not the full amount.

Suggested change:

```solidity
uint256 grossSubsidy = _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount);
uint256 processingFee = _storage.fees.processingFeePercentage.calculateRate(grossSubsidy);
uint256 subsidyMinusFees = grossSubsidy - processingFee;
```

## Proof of Concept

The following PoC demonstrates that `processingFeePercentage` is applied to the full LONG amount instead of to the subsidy amount during LONG payments, causing venue underpayment.

Save as `poc-processing-fee-on-subsidy.test.ts` and run with: pnpm test test/v2/platform/poc-processing-fee-on-subsidy.test.ts -s

TypeScript PoC:

```typescript
import { ethers, network } from 'hardhat';
import { BigNumber } from 'ethers';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import EthCrypto from 'eth-crypto';
import { expect } from 'chai';

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
import { CustomerInfoStruct, VenueInfoStruct, VenueRulesStruct } from '../../../typechain-types/contracts/v2/platform/BelongCheckIn';
import { deployHelper, deploySignatureVerifier } from '../../../helpers/deployLibraries';
import {
  deployBelongCheckIn,
  deployCreditTokens,
  deployEscrow,
  deployFactory,
  deployRoyaltiesReceiverV2Implementation,
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployStaking,
  deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';
import { deployMockTransferValidatorV2, deployPriceFeeds } from '../../../helpers/deployMockFixtures';
import { getSignerFromAddress, getToken } from '../../../helpers/fork';
import { ChainIds, chainRPCs } from '../../../utils/chain-ids';
import { U, u } from '../../../helpers/math';

// This PoC demonstrates that processingFeePercentage is applied to the full LONG amount
// instead of to the subsidy amount during LONG payments, underpaying the venue.

describe('PoC: processingFee applied to full LONG instead of subsidy', () => {
  const chainId = 31337; // hardhat local

  // Mainnet artifacts (same as existing tests)
  const WETH_ADDRESS = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2';
  const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';
  const ENA_ADDRESS = '0x57e114B691Db790C35207b2e685D4A43181e6061'; // used instead of LONG

  const USDC_WHALE_ADDRESS = '0x8EB8a3b98659Cce290402893d0123abb75E3ab28';
  const ENA_WHALE_ADDRESS = '0xF977814e90dA44bFA03b6295A0616a897441aceC';

  const UNISWAP_FACTORY_ADDRESS = '0x1F98431c8aD98523631AE4a59f267346ea31F984';
  const UNISWAP_ROUTER_ADDRESS = '0xE592427A0AEce92De3Edee1F18E0157C05861564';
  const UNISWAP_QUOTER_ADDRESS = '0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6';
  const POOL_FEE = 3000;
  const MAX_PRICEFEED_DELAY = 3600;

  before(async () => {
    await network.provider.request({
      method: 'hardhat_reset',
      params: [
        {
          forking: {
            jsonRpcUrl: chainRPCs(ChainIds.mainnet),
            enable: true,
          },
        },
      ],
    });
  });

  async function fixture() {
    const [admin, treasury, manager, minter, burner, pauser, referral] = await ethers.getSigners();
    const signer = EthCrypto.createIdentity();

    const USDC_whale = await getSignerFromAddress(USDC_WHALE_ADDRESS);
    const ENA_whale = await getSignerFromAddress(ENA_WHALE_ADDRESS);

    const USDC = await getToken(USDC_ADDRESS);
    const ENA = await getToken(ENA_ADDRESS);

    const signatureVerifier: SignatureVerifier = await deploySignatureVerifier();
    const validator: MockTransferValidatorV2 = await deployMockTransferValidatorV2();
    const accessTokenImplementation: AccessToken = await deployAccessTokenImplementation(signatureVerifier.address);
    const royaltiesReceiverV2Implementation: RoyaltiesReceiverV2 = await deployRoyaltiesReceiverV2Implementation();
    const creditTokenImplementation: CreditToken = await deployCreditTokenImplementation();
    const vestingWallet: VestingWalletExtended = await deployVestingWalletImplementation();

    const implementations: Factory.ImplementationsStruct = {
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

    const belongCheckIn: BelongCheckIn = await deployBelongCheckIn(
      signatureVerifier.address,
      helper.address,
      admin.address,
      {
        swapPoolFees: POOL_FEE,
        slippageBps: BigNumber.from(10).pow(27).sub(1),
        swapV3Factory: UNISWAP_FACTORY_ADDRESS,
        swapV3Router: UNISWAP_ROUTER_ADDRESS,
        swapV3Quoter: UNISWAP_QUOTER_ADDRESS,
        wNativeCurrency: WETH_ADDRESS,
        usdc: USDC_ADDRESS,
        long: ENA_ADDRESS,
        maxPriceFeedDelay: MAX_PRICEFEED_DELAY,
      },
    );

    const escrow: Escrow = await deployEscrow(belongCheckIn.address);
    const { pf1 } = await deployPriceFeeds();

    // Wire contracts
    const { venueToken, promoterToken } = await deployCreditTokens(
      true,
      false,
      factory.address,
      signer.privateKey,
      admin,
      admin.address,
      belongCheckIn.address,
      belongCheckIn.address,
    );
    await belongCheckIn.setContracts({
      factory: factory.address,
      escrow: escrow.address,
      staking: staking.address,
      venueToken: venueToken.address,
      promoterToken: promoterToken.address,
      longPF: pf1.address,
    });

    return {
      admin,
      treasury,
      referral,
      signer,
      belongCheckIn,
      escrow,
      helper,
      USDC,
      ENA,
      USDC_whale,
      ENA_whale,
    };
  }

  it('demonstrates underpayment due to fee-on-full instead of fee-on-subsidy', async () => {
    const { belongCheckIn, escrow, helper, USDC, ENA, USDC_whale, ENA_whale, signer } = await loadFixture(fixture);

    // Prepare a venue deposit (LONG-only payments)
    const uri = 'uri://venue';
    const venueAmount = await u(100, USDC); // $100
    const venue = USDC_whale.address;
    const venueMessage = ethers.utils.solidityKeccak256(
      ['address', 'bytes32', 'string', 'uint256'],
      [venue, ethers.constants.HashZero, uri, chainId],
    );
    const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);

    const venueInfo: VenueInfoStruct = {
      rules: { paymentType: 2, bountyType: 0, longPaymentType: 0 } as VenueRulesStruct, // LONG only
      venue,
      amount: venueAmount,
      referralCode: ethers.constants.HashZero,
      uri,
      signature: venueSignature,
    };

    // Approve and deposit (includes $5 convenience fee swapped to LONG into escrow)
    const convenienceFeeAmount = U(5, 6);
    await USDC.connect(USDC_whale).approve(belongCheckIn.address, convenienceFeeAmount.add(venueAmount));
    await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

    // Customer LONG payment with no promoter
    const customerAmount = ethers.utils.parseEther('5'); // 5 LONG to keep escrow solvent against subsidy pulls
    const customerMessage = ethers.utils.solidityKeccak256(
      ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
      [
        false, // paymentInUSDC
        0, // visitBountyAmount
        0, // spendBountyPercentage
        ENA_whale.address, // customer
        USDC_whale.address, // venueToPayFor
        ethers.constants.AddressZero, // promoter
        customerAmount, // amount
        chainId,
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

    // Snapshot balances
    const venueBalance_before = await ENA.balanceOf(USDC_whale.address);

    // Execute payment
    await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);

    const venueBalance_after = await ENA.balanceOf(USDC_whale.address);
    const actualVenueReceived = venueBalance_after.sub(venueBalance_before);

    // Fetch fee parameters
    const fees = (await belongCheckIn.belongCheckInStorage()).fees;

    // Components
    const grossSubsidy = await helper.calculateRate(fees.platformSubsidyPercentage, customerAmount);
    const discount = await helper.calculateRate(fees.longCustomerDiscountPercentage, customerAmount);
    const fromCustomer = BigNumber.from(customerAmount).sub(discount);

    // Buggy behavior (processing fee applied to full amount)
    const processingOnFull = await helper.calculateRate(fees.processingFeePercentage, customerAmount);
    const buggyFromEscrow = BigNumber.from(grossSubsidy).sub(processingOnFull);
    const buggyExpectedVenue = buggyFromEscrow.add(fromCustomer);

    // Correct behavior (processing fee applied to subsidy)
    const processingOnSubsidy = await helper.calculateRate(fees.processingFeePercentage, grossSubsidy);
    const correctFromEscrow = BigNumber.from(grossSubsidy).sub(processingOnSubsidy);
    const correctExpectedVenue = correctFromEscrow.add(fromCustomer);

    // Pretty logging for visibility
    const longDecimals = await ENA.decimals();
    const fmt = (bn: BigNumber) => ethers.utils.formatUnits(bn, longDecimals);
    const bpsToNum = (v: any) => (v && typeof v === 'object' && 'toNumber' in v ? v.toNumber() : Number(v));
    const bpsFmt = (bps: any) => `${String(bps)} bps (${(bpsToNum(bps) / 100).toFixed(2)}%)`;
    const shortfall = BigNumber.from(processingOnFull).sub(processingOnSubsidy);
    const shortfallBpsOfFull = shortfall.mul(10000).div(customerAmount);

    // Scenario breakdown
    // Raw = wei, Pretty = token units (18 decimals)
    // Note: ENA is used as LONG stand-in, assumed 18 decimals
    // eslint-disable-next-line no-console
    console.log('--- LONG payment PoC breakdown ---');
    // eslint-disable-next-line no-console
    console.log('customerAmount  (raw, pretty):', customerAmount.toString(), fmt(customerAmount));
    // eslint-disable-next-line no-console
    console.log('fees:', {
      platformSubsidyPercentage: bpsFmt(fees.platformSubsidyPercentage),
      processingFeePercentage: bpsFmt(fees.processingFeePercentage),
      longCustomerDiscountPercentage: bpsFmt(fees.longCustomerDiscountPercentage),
    });
    // eslint-disable-next-line no-console
    console.log('grossSubsidy    (raw, pretty):', grossSubsidy.toString(), fmt(grossSubsidy));
    // eslint-disable-next-line no-console
    console.log('discount       (raw, pretty):', discount.toString(), fmt(discount));
    // eslint-disable-next-line no-console
    console.log('fromCustomer   (raw, pretty):', fromCustomer.toString(), fmt(fromCustomer));
    // eslint-disable-next-line no-console
    console.log('processingFull  (raw, pretty):', processingOnFull.toString(), fmt(processingOnFull));
    // eslint-disable-next-line no-console
    console.log('processingSubs  (raw, pretty):', processingOnSubsidy.toString(), fmt(processingOnSubsidy));
    // eslint-disable-next-line no-console
    console.log('buggyFromEscrow (raw, pretty):', buggyFromEscrow.toString(), fmt(buggyFromEscrow));
    // eslint-disable-next-line no-console
    console.log('correctFromEsc. (raw, pretty):', correctFromEscrow.toString(), fmt(correctFromEscrow));
    // eslint-disable-next-line no-console
    console.log('buggyVenueTotal (raw, pretty):', buggyExpectedVenue.toString(), fmt(buggyExpectedVenue));
    // eslint-disable-next-line no-console
    console.log('correctVenueTot (raw, pretty):', correctExpectedVenue.toString(), fmt(correctExpectedVenue));
    // eslint-disable-next-line no-console
    console.log('actualVenueRecv (raw, pretty):', actualVenueReceived.toString(), fmt(actualVenueReceived));
    // eslint-disable-next-line no-console
    console.log('shortfall       (raw, pretty):', shortfall.toString(), fmt(shortfall));
    // eslint-disable-next-line no-console
    console.log('shortfall vs full amount:', `${shortfallBpsOfFull.toString()} bps`, `(${(shortfallBpsOfFull.toNumber() / 100).toFixed(3)}%)`);

    // PoC assertions
    // 1) Actual matches buggy formula
    expect(actualVenueReceived).to.eq(buggyExpectedVenue);
    // 2) Actual is strictly less than correct amount (venue underpaid)
    expect(correctExpectedVenue).to.gt(actualVenueReceived);
    // 3) The shortfall equals processingOnFull - processingOnSubsidy
    const expectedShortfall = BigNumber.from(processingOnFull).sub(processingOnSubsidy);
    expect(correctExpectedVenue.sub(actualVenueReceived)).to.eq(expectedShortfall);
  });
});
```

<details>

<summary>Logs from PoC (5 LONG payment; default fees)</summary>

```
--- LONG payment PoC breakdown ---
customerAmount  (raw, pretty): 5000000000000000000 5.0
fees: {
  platformSubsidyPercentage: '300 bps (3.00%)',
  processingFeePercentage: '250 bps (2.50%)',
  longCustomerDiscountPercentage: '300 bps (3.00%)'
}
grossSubsidy    (raw, pretty): 150000000000000000 0.15
discount       (raw, pretty): 150000000000000000 0.15
fromCustomer   (raw, pretty): 4850000000000000000 4.85
processingFull  (raw, pretty): 125000000000000000 0.125
processingSubs  (raw, pretty): 3750000000000000 0.00375
buggyFromEscrow (raw, pretty): 25000000000000000 0.025
correctFromEsc. (raw, pretty): 146250000000000000 0.14625
buggyVenueTotal (raw, pretty): 4875000000000000000 4.875
correctVenueTot (raw, pretty): 4996250000000000000 4.99625
actualVenueRecv (raw, pretty): 4875000000000000000 4.875
shortfall       (raw, pretty): 121250000000000000 0.12125
shortfall vs full amount: 242 bps (2.420%)
    ✔ demonstrates underpayment due to fee-on-full instead of fee-on-subsidy (143231ms)


  1 passing (3m)
```

As seen in the logs:

* Venue receives 4.875 LONG (buggy).
* Correct would be 4.99625 LONG.
* Shortfall is 0.12125 LONG (≈ 2.42% of full amount), matching “processing on full minus processing on subsidy.”

</details>
