# 57398 sc critical incorrect platform subsidy processing in long payments causing venue payout failures

**Submitted on Oct 25th 2025 at 20:13:27 UTC by @Oxv1bh4 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57398
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

The `payToVenue()` function in `BelongCheckIn.sol` can fail when customers pay in LONG tokens. The venue’s subsidy is intended to be paid from the protocol’s escrow that is being funded by convenience fees. However, if the subsidy amount exceeds the available LONG balance in escrow, the transaction will revert. This prevents venues from receiving payments in LONG. The documentation states that **Treasury must purchase LONG from market to fulfill payment**

### Vulnerability Details

When a venue makes a deposit via `BelongCheckIn.sol::venueDeposit`, a convenience fee (e.g., $5) is charged. This fee is converted into LONG tokens and tracked as the venue's LONG balance in the escrow through `Escrow.sol::venueDeposit`. Later, when a customer pays the venue in LONG tokens through `BelongCheckIn.sol::payVenue`, the platform provides a subsidy to the venue. This subsidy is drawn from the venue's LONG balance in the escrow via `Escrow.sol::distributeLONGDiscount`.

If the venue’s escrow balance is insufficient to cover the subsidy, the customer’s payment in LONG tokens will revert. However, according to Belong’s documentation, the platform is expected to purchase LONG from the market to ensure the payment is fulfilled.

### Impact Details

If the venue’s escrow lacks sufficient LONG tokens to cover the platform subsidy, any customer payment in LONG will fail. This can prevent venues from receiving payments and block normal platform operations. According to the Belong documentation, the platform is expected to purchase LONG from the market to fulfill such payments. Failure to do so effectively makes the protocol unable to meet its financial obligations, which constitutes a form of **protocol insolvency**, as the system cannot honor its commitments to users and venues.

## References

* **BelongCheckIn.sol::venueDeposit :** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol?utm\\_source=immunefi#L379-L426>
* **BelongCheckIn.sol::payToVenue :** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol#L435-L506>
* **Escrow\.sol::venueDeposit :** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Escrow.sol#L100-L106>
* **Escrow\.sol::distributeLONGDiscount :** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Escrow.sol#L113-L126>

## Proof of Concept

Update the test case `payToVenue() (w/o promoter)` in the `test/v2/platform/belong-check-in.test.ts` as follows:

{% code title="test snippet" %}

```
```

{% endcode %}

```solidity
it.only('payToVenue() (w/o promoter)', async () => {
      const { belongCheckIn, escrow, helper, signer, USDC, ENA, USDC_whale, ENA_whale } = await loadFixture(fixture);
    
      // Venue setup
      const uri = 'uriuri';
      const venueAmount = await u(100, USDC);
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
    
      const escrowDeposit = await escrow.venueDeposits(USDC_whale.address);
      console.log('LONG deposits of the venue in the Escrow: ', escrowDeposit.longDeposits);
    
      // Customer setup
      const customerAmount = ethers.utils.parseEther('2000');
      console.log('Customer amount: ', customerAmount);
    
      const customerMessage = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [
          false,                  // paymentInUSDC
          0,                      // visitBountyAmount
          0,                      // spendBountyPercentage
          ENA_whale.address,      // customer
          USDC_whale.address,     // venueToPayFor
          ethers.constants.AddressZero, // promoter
          customerAmount,         // amount
          chainId,                // block.chainid
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
    
      // Calculate subsidy and fees
      const platformSubsidy = await helper.calculateRate(
        (await belongCheckIn.belongCheckInStorage()).fees.platformSubsidyPercentage,
        customerAmount,
      );
    
      const processingFee = await helper.calculateRate(
        (await belongCheckIn.belongCheckInStorage()).fees.processingFeePercentage,
        customerAmount,
      );
    
      const fromEscrowToVenue = platformSubsidy.sub(processingFee);
      console.log('The amount of LONG to be transferred from escrow to venue: ', fromEscrowToVenue);
    
      // Approve LONG
      await ENA.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);
    
      // Expect revert due to insufficient escrow LONG balance
      await expect(
        belongCheckIn.connect(ENA_whale).payToVenue(customerInfo)
      ).to.revertedWithCustomError(escrow, 'NotEnoughLONGs');
    });
```

Run the command: `npm run test`.

## Attack path

{% stepper %}
{% step %}
Venue deposits 100 USDC, with a platform fee of 10% and a convenience fee of 5 USDC.
{% endstep %}

{% step %}
The 5 USDC convenience fee is converted to LONG via a Uniswap V3 swap, resulting in \~8 LONG credited to the venue’s escrow balance.
{% endstep %}

{% step %}
A customer pays the venue in LONG tokens (e.g., 2000 LONG). If the platform subsidy is calculated as 10 LONG, the escrow does not have enough LONG to cover it, causing the payment to fail.
{% endstep %}
{% endstepper %}
