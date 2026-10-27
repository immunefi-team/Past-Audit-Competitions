# 57829 sc high incorrect fee implementation in paytovenue long payment path causes protocol fees to be permanently locked in escrow

**Submitted on Oct 29th 2025 at 05:11:03 UTC by @chupinexx for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57829
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties
  * Protocol insolvency
  * Unclaimed platform fees lead to protocol's fees lock

## Description

### Brief / Intro

The `payToVenue` function's logic for processing LONG token payments contains a critical accounting error. While the code correctly calculates the intended processing fee by subtracting it from a platform subsidy, it fails to execute the subsequent transfer of this fee to the protocol's treasury. Consequently, the fee is never collected; instead, it remains stranded within the venue's LONG balance in the Escrow contract. This would result in a permanent and irrecoverable loss of all processing fee revenue generated from every LONG payment across the entire platform.

### Vulnerability Details

The issue is a straightforward but critical omission in the fund transfer logic. The protocol intends to collect a 2.5% fee on LONG payments, as stated in the documentation, but the implementation fails to complete the final step of actually collecting the fee.

According to the project's documentation, a `2.5%` fee is supposed to be collected on customer payments made with the LONG token:

<https://belongnet.github.io/docs/belong-checkin/whitepaper#venue-fees>

> Venue Fees\
> LONG Payment Processing: 2.5% (Lower than cards, on customer payments)

The code correctly performs the calculation of this fee. When a customer pays with LONG, the contract calculates a platform subsidy and then subtracts the processing fee from it.

```solidity
// contracts/v2/platform/BelongCheckIn.sol -> payToVenue function

// The processing fee is calculated here...
uint256 subsidyMinusFees =
    _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount)
    - _storage.fees.processingFeePercentage.calculateRate(customerInfo.amount);

// ...but only the *result* of the subtraction is used.
_storage.contracts.escrow
    .distributeLONGDiscount(customerInfo.venueToPayFor, address(this), subsidyMinusFees);
```

This is the core of the flaw. The code calculates a value for the processing fee, but this value is only used to decrease the amount of the subsidy being pulled from Escrow. The fee amount itself is never transferred out of the Escrow contract to the protocol's treasury (which would typically be handled by a call to `_handleRevenue`).

### Tracing the Locked Funds: A Step-by-Step Example

{% stepper %}
{% step %}

### Scenario Setup

* Customer gross payment: 100 LONG
* `platformSubsidyPercentage`: 3% → 3 LONG
* `processingFeePercentage`: 2.5% → 2.5 LONG
  {% endstep %}

{% step %}

### Execution Flow

1. `subsidyMinusFees` is calculated: `3 LONG - 2.5 LONG = 0.5 LONG`.
2. `_storage.contracts.escrow.distributeLONGDiscount(...)` is called and pulls **0.5 LONG** from the venue's LONG balance in Escrow.
3. The customer pays their discounted portion; funds are routed to the venue.
   {% endstep %}

{% step %}

### Result / Problem

* Protocol intended: provide 0.5 LONG subsidy and collect 2.5 LONG fee.
* Actual behavior: only 0.5 LONG is pulled from the venue's escrowed subsidy balance; the 2.5 LONG processing fee is never transferred and remains in the Escrow contract under the venue's LONG balance.
* There is no function in the contract to withdraw these residual fee amounts from the venue's escrow balance, so they are permanently locked and unrecoverable by the protocol.
  {% endstep %}
  {% endstepper %}

### Impact Details

* Direct and Permanent Financial Loss: The protocol will lose **2.5% of the gross value of every LONG token payment**. These funds become irrecoverably locked in the Escrow contract. Over time, this results in significant lost revenue.

## References

* Vulnerable code location:\
  <https://github.com/belongnet/checkin-contracts/blob/6b78ead6186c49cfec2787522460ddd516579a6b/contracts/v2/platform/BelongCheckIn.sol#L473-L484>
* Documentation:\
  <https://belongnet.github.io/docs/belong-checkin/whitepaper#venue-fees>

## Proof of Concept

Add this PoC test to the existing test suite in `test/v2/platform/belong-check-in.test.ts`. The PoC tracks the platform address balance (obtained from `factory.nftFactoryParameters().platformAddress`) and demonstrates that processing fees are never received. For every LONG payment, 2.5% of the transaction value is permanently locked in escrow.

```rust
it('PoC: Platform loses processing fees on LONG payments - fees remain locked in escrow', async () => {  
  const { belongCheckIn, escrow, helper, signer, factory, USDC, ENA, USDC_whale, ENA_whale } = await loadFixture(fixture);  
  
 
  // STEP 1: Setup - Venue deposits USDC  
 
  console.log('\n=== STEP 1: Venue Deposit ===');  
  const uri = 'test-venue-uri';  
  const venueAmount = await u(100, USDC);  
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
    
  const willBeTaken = convenienceFeeAmount.add(venueAmount);  
  await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);  
  await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);  
  console.log('Venue deposited:', ethers.utils.formatUnits(venueAmount, 6), 'USDC');  
  
  
  // STEP 2: Get platform address and initial balances  
  
  console.log('\n=== STEP 2: Record Initial Balances ===');  
  const platformAddress = (await factory.nftFactoryParameters()).platformAddress;  
  console.log('Platform address:', platformAddress);  
    
  const platformBalance_LONG_before = await ENA.balanceOf(platformAddress);  
  const escrowBalance_LONG_before = await ENA.balanceOf(escrow.address);  
  const belongCheckInBalance_LONG_before = await ENA.balanceOf(belongCheckIn.address);  
    
  console.log('Platform LONG balance BEFORE:', ethers.utils.formatEther(platformBalance_LONG_before));  
  console.log('Escrow LONG balance BEFORE:', ethers.utils.formatEther(escrowBalance_LONG_before));  
  console.log('BelongCheckIn LONG balance BEFORE:', ethers.utils.formatEther(belongCheckInBalance_LONG_before));  
  
   
  // STEP 3: Customer pays with LONG  
   
  console.log('\n=== STEP 3: Customer Pays with LONG ===');  
  const customerAmount = ethers.utils.parseEther('10'); // 10 LONG tokens  
  const customerMessage = ethers.utils.solidityKeccak256(  
    ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],  
    [  
      false, // paymentInUSDC = false (paying with LONG)  
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
  
  // Calculate expected fees  
  const storage = await belongCheckIn.belongCheckInStorage();  
  const platformSubsidy = await helper.calculateRate(storage.fees.platformSubsidyPercentage, customerAmount);  
  const processingFee = await helper.calculateRate(storage.fees.processingFeePercentage, customerAmount);  
  const longCustomerDiscount = await helper.calculateRate(storage.fees.longCustomerDiscountPercentage, customerAmount);  
    
  console.log('Customer payment amount:', ethers.utils.formatEther(customerAmount), 'LONG');  
  console.log('Platform subsidy (from escrow):', ethers.utils.formatEther(platformSubsidy), 'LONG');  
  console.log('Processing fee (2.5%):', ethers.utils.formatEther(processingFee), 'LONG');  
  console.log('Customer discount (3%):', ethers.utils.formatEther(longCustomerDiscount), 'LONG');  
  
  await ENA.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);  
  await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);  
  console.log(' Customer payment completed');  
  
  
  // STEP 4: Check balances after payment  
  
  console.log('\n=== STEP 4: Verify Platform Did NOT Receive Fees ===');  
  const platformBalance_LONG_after = await ENA.balanceOf(platformAddress);  
  const escrowBalance_LONG_after = await ENA.balanceOf(escrow.address);  
  const belongCheckInBalance_LONG_after = await ENA.balanceOf(belongCheckIn.address);  
    
  console.log('Platform LONG balance AFTER:', ethers.utils.formatEther(platformBalance_LONG_after));  
  console.log('Escrow LONG balance AFTER:', ethers.utils.formatEther(escrowBalance_LONG_after));  
  console.log('BelongCheckIn LONG balance AFTER:', ethers.utils.formatEther(belongCheckInBalance_LONG_after));  
  
  
  // STEP 5: Calculate where the processing fee went  
 
  console.log('\n=== STEP 5: Analysis ===');  
  const platformBalanceChange = platformBalance_LONG_after.sub(platformBalance_LONG_before);  
  const escrowBalanceChange = escrowBalance_LONG_before.sub(escrowBalance_LONG_after);  
  const fromEscrowToVenue = platformSubsidy.sub(processingFee);  
    
  console.log('Platform balance change:', ethers.utils.formatEther(platformBalanceChange), 'LONG');  
  console.log('Expected platform fee:', ethers.utils.formatEther(processingFee), 'LONG');  
  console.log('Escrow released:', ethers.utils.formatEther(escrowBalanceChange), 'LONG');  
  console.log('Expected escrow release (subsidy - fee):', ethers.utils.formatEther(fromEscrowToVenue), 'LONG');  
  
   
  // STEP 6: Assertions - Prove the vulnerability  
   
  console.log('\n=== STEP 6: Vulnerability Proof ===');  
    
// Issue: Platform balance should have increased by processingFee, but it didn't  
  expect(platformBalanceChange).to.equal(0);  
  console.log(' issue CONFIRMED: Platform received 0 LONG (expected', ethers.utils.formatEther(processingFee), 'LONG)');  
    
  // The processing fee is stuck in escrow  
  const expectedEscrowBalance = escrowBalance_LONG_before.sub(fromEscrowToVenue);  
  expect(escrowBalance_LONG_after).to.equal(expectedEscrowBalance);  
  console.log(' Processing fee remains locked in escrow:', ethers.utils.formatEther(processingFee), 'LONG');  
    
  // Calculate total loss  
  console.log('\n=== IMPACT SUMMARY ===');  
  console.log('Per-transaction loss:', ethers.utils.formatEther(processingFee), 'LONG (2.5% of', ethers.utils.formatEther(customerAmount), 'LONG)');  
  console.log('This fee is permanently locked in escrow with no recovery mechanism');  
  console.log('Platform receives 0% instead of the documented 2.5% processing fee');  
});
```

Run the PoC:

```bash
# Run the specific PoC test  
npx hardhat test --grep "PoC: Platform loses processing fees on LONG payments - fees remain locked in escrow"
```

The PoC shows:

* Platform LONG balance remains unchanged (no increase)
* Processing fee (2.5% of payment) remains locked in escrow
* Platform receives 0% instead of the documented 2.5% processing fee
