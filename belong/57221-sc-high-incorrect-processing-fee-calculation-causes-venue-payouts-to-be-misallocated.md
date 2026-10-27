# 57221 sc high incorrect processing fee calculation causes venue payouts to be misallocated

**Submitted on Oct 24th 2025 at 14:06:45 UTC by @Oxv1bh4 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57221
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:** Permanent freezing of unclaimed yield

## Description

### Brief / Intro

The `processingFeePercentage` is intended to be applied as a fraction of the `platformSubsidyPercentage` when customers pay with LONG tokens. Due to a logic error, it is instead computed on the total payment amount rather than on the subsidy, causing venues to receive substantially fewer LONG tokens than intended.

### Vulnerability Details

In the `BelongCheckIn.sol::payToVenue` function (link below), when customers pay with LONG tokens:

* `platformSubsidyPercentage` is the LONG subsidy the platform adds for the merchant.
* `processingFeePercentage` is the portion of that LONG subsidy collected by the platform as a processing fee.

However, the function calculates the processing fee on the total LONG payment amount instead of on the subsidy amount.

Relevant code excerpt:

```solidity
// platform subsidy - processing fee
// @audit - Processing Fee should be calculated on the Platform Subsidy amount.
uint256 subsidyMinusFees =
    _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount)
    - _storage.fees.processingFeePercentage.calculateRate(customerInfo.amount);
_storage.contracts.escrow
    .distributeLONGDiscount(customerInfo.venueToPayFor, address(this), subsidyMinusFees);
```

This subtracts processing fees calculated over the entire customer payment (`customerInfo.amount`) instead of the platform subsidy portion.

## Impact Details

Affected party: Venue / Merchant

Because `processingFeePercentage` is applied on the total payment rather than on the platform subsidy, venues receive less LONG than intended.

Example:

* Payment: 1,000 LONG
* Customer Discount: 5%
* Platform Subsidy: 3%
* Processing Fee: 2.5%
* Correct payout: 979.25 LONG
* Current payout: 955 LONG
* Loss per payment: 24.25 LONG

Repeated across many payments, this yields cumulative financial loss and reduced venue revenue.

Severity: High — aligns with permanent freezing of unclaimed yield.

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol?utm\\_source=immunefi#L184>
* <https://github.com/belongnet/checkin-contracts/blob/main/docs/guides/BelongCheckinOverview.md?plain=1#L79>

## Proof of Concept

The reporter updated the test case `payToVenue() (both payment) (w/o promoter)` in `test/v2/platform/belong-check-in.test.ts` to demonstrate the difference between the expected escrow-to-venue transfer (where processing fee is applied to the subsidy) and the actual transfer (where processing fee is incorrectly applied to the total amount).

Test snippet:

```ts
it.only('payToVenue() (both payment) (w/o promoter)', async () => {
      const { belongCheckIn, escrow, helper, signer, USDC, ENA, USDC_whale, ENA_whale } = await loadFixture(fixture);

      const uri = 'uriuri';
      const venueAmount = await u(100, USDC);
      const venue = USDC_whale.address;
      const venueMessage = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 3, bountyType: 0, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature: venueSignature,
      };
      const willBeTaken = convenienceFeeAmount.add(venueAmount);
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      const customerAmount = ethers.utils.parseEther('5');
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

      const platformSubsidy = await helper.calculateRate(
        (
          await belongCheckIn.belongCheckInStorage()
        ).fees.platformSubsidyPercentage,
        customerAmount,
      );
      const processingFee = await helper.calculateRate(
        (
          await belongCheckIn.belongCheckInStorage()
        ).fees.processingFeePercentage,
        platformSubsidy,
      );
      const fromEscrowToVenueExpected = platformSubsidy.sub(processingFee);

      await ENA.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);

      const escrowBalance_before = await ENA.balanceOf(escrow.address);

      await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);

      const escrowBalance_after = await ENA.balanceOf(escrow.address);

      const fromEscrowToVenueActual = escrowBalance_before.sub(escrowBalance_after);

      expect(fromEscrowToVenueExpected).to.not.eq(fromEscrowToVenueActual);

      const lossAmount = fromEscrowToVenueExpected.sub(fromEscrowToVenueActual);

      console.log("Expected LONG amount to be paid from the escrow to the venue: ", fromEscrowToVenueExpected);
      console.log("Actual LONG amount to be paid from the escrow to the venue: ", fromEscrowToVenueActual);
      console.log("Loss: ",lossAmount);
    });
```

Run the test with: `npm run test`

***

Note: Links and code snippets are preserved from the original report.
