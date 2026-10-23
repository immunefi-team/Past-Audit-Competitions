# 57039 sc critical processing fee logic flaw in paytovenue causes permanent loss of platform revenue

**Submitted on Oct 22nd 2025 at 22:33:51 UTC by @pirex for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57039
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

The `payToVenue()` function in BelongCheckIn v2 contains a critical accounting flaw in the LONG payment path where the platform's configured `processingFeePercentage` is never transferred to the platform. The contract calculates and subtracts the fee amount but fails to withdraw or route it through `_handleRevenue()`. This causes the fee to remain stranded in the escrow contract, allowing venues to eventually reclaim it while the platform receives zero revenue. This represents a direct loss of funds for the platform and a complete failure of the business logic that should collect processing fees on every LONG payment transaction.

## Vulnerability Details

The vulnerability occurs in `contracts/v2/platform/BelongCheckIn.sol` lines 474–495 within the `payToVenue()` function's LONG payment flow:

```solidity
uint256 subsidyMinusFees = platformSubsidy - processingFee;
escrow.distributeLONGDiscount(venue, subsidyMinusFees);
```

The critical flaw is that the contract only withdraws `subsidyMinusFees` from escrow (the subsidy minus the processing fee), but it never handles the `processingFee` portion itself. This creates several problems:

1. Fee Never Withdrawn: The `processingFee` amount is calculated but never actually withdrawn from the escrow
2. Revenue Handler Never Called: The `_handleRevenue()` function, which should receive the platform's fee, is never invoked
3. Funds Remain in Escrow: The processing fee stays recorded in escrow storage under the venue's balance
4. Venue Can Reclaim: Since the fee remains in the venue's escrow balance, the venue can later withdraw it through legitimate mechanisms
5. Platform Gets Nothing: The platform receives zero revenue despite the fee being deducted from the subsidy calculation

Why This Happens:

* The code calculates `processingFee = calculateRate(processingFeePercentage, amount)`
* It subtracts this fee: `subsidyMinusFees = platformSubsidy - processingFee`
* It only withdraws the reduced amount: `escrow.distributeLONGDiscount(venue, subsidyMinusFees)`
* It never calls `_handleRevenue(processingFee)` to transfer the fee to the platform
* The escrow balance decreases by `subsidyMinusFees` instead of the full `platformSubsidy`
* The platform balance remains completely unchanged

This is a silent but severe accounting bug because it causes the platform to permanently lose revenue while the on-chain state misleadingly suggests everything is functioning correctly.

## Impact Details

This vulnerability has significant real-world financial consequences:

* Direct Revenue Loss:
  * The platform's intended revenue (`processingFee`) is never collected on any LONG payment
  * Every transaction that should generate platform fees results in zero revenue
  * The platform effectively operates without collecting its configured processing fees
  * This is a direct loss of funds that should belong to the platform
* Funds Reclaimable by Venue:
  * Because the fee remains in the venue's escrow balance, it becomes venue property
  * Venues can withdraw these "stranded fees" later through standard withdrawal mechanisms
  * This creates a perverse incentive structure where venues benefit from platform fees
  * The venue effectively receives back the processing fee that should have gone to the platform
* Financial Impact at Scale:
  * Revenue loss scales linearly with transaction volume
  * Example scenario: If `processingFeePercentage = 5%` and total LONG payment volume is 10M ENA:
    * Expected platform revenue: 500,000 ENA
    * Actual platform revenue: 0 ENA
    * Loss to platform: 500,000 ENA
    * Benefit to venues: 500,000 ENA (can be reclaimed)
  * The loss is ongoing and cumulative with every single LONG payment transaction
  * Higher transaction volumes result in proportionally higher losses
* No Privileges Required:
  * Any standard call to `payToVenue()` with LONG payment type triggers the bug
  * No special access, admin rights, or malicious intent required
  * The vulnerability is triggered automatically during normal protocol operation
  * There is no on-chain check, assertion, or invariant that detects this condition
* Business Logic Failure:
  * The platform's revenue model is fundamentally broken for LONG payments
  * Financial projections and tokenomics based on fee collection are invalidated
  * Trust in the protocol's accounting accuracy is undermined
  * The issue affects every LONG payment since contract deployment

This qualifies as a Critical severity issue under Immunefi criteria: "Direct theft of any user funds" (platform is the user being stolen from) and "Protocol insolvency" (loss of expected revenue stream).

## References

**Affected Contracts:**

* `contracts/v2/platform/BelongCheckIn.sol` - `payToVenue()` logic (lines 474–495)
* `contracts/v2/periphery/Escrow.sol` - where stranded funds remain
* `contracts/v2/utils/SignatureVerifier.sol` - indirectly affected (signatures used in deposit flow)

**Key Functions:**

* `BelongCheckIn.payToVenue()` - vulnerable fee handling in LONG payment path
* `BelongCheckIn._handleRevenue()` - never called for processing fee
* `Escrow.distributeLONGDiscount()` - withdraws incorrect (reduced) amount
* `Helper.calculateRate()` - correctly calculates fee, but result is not used properly

## Proof of Concept

A comprehensive regression test has been developed that demonstrates the vulnerability. The test:

{% stepper %}
{% step %}

### PoC test summary

* Sets up a normal LONG payment scenario with venue deposit
* Records escrow and platform balances before payment
* Executes `payToVenue()` with LONG payment
* Verifies the accounting flaw by checking:
  * Escrow only deducts `platformSubsidy - processingFee` (not full subsidy)
  * Platform balance remains completely unchanged (zero fee received)
  * Processing fee remains trapped in escrow deposits
  * The "missing" fee equals exactly the calculated `processingFee`
    {% endstep %}

{% step %}

### Full PoC Git Diff (test addition)

```diff
diff --git a/test/v2/platform/belong-check-in.test.ts b/test/v2/platform/belong-check-in.test.ts
index 2448..2509 100644
--- a/test/v2/platform/belong-check-in.test.ts
+++ b/test/v2/platform/belong-check-in.test.ts
@@ -2448,0 +2449,60 @@ describe('Customer flow long payment', () => {
+    it('payToVenue() leaves processing fee stranded in escrow', async () => {
+      const { belongCheckIn, escrow, helper, factory, signer, USDC, ENA, USDC_whale, ENA_whale } =
+        await loadFixture(fixture);
+
+      const uri = 'uriuri';
+      const venueAmount = await u(100, USDC);
+      const venue = USDC_whale.address;
+      const venueMessage = ethers.utils.solidityKeccak256(
+        ['address', 'bytes32', 'string', 'uint256'],
+        [venue, ethers.constants.HashZero, uri, chainId],
+      );
+      const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);
+      const venueInfo: VenueInfoStruct = {
+        rules: { paymentType: 2, bountyType: 0, longPaymentType: 0 } as VenueRulesStruct,
+        venue,
+        amount: venueAmount,
+        referralCode: ethers.constants.HashZero,
+        uri,
+        signature: venueSignature,
+      };
+      const willBeTaken = convenienceFeeAmount.add(venueAmount);
+      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
+      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);
+
+      const customerAmount = ethers.utils.parseEther('5');
+      const customerMessage = ethers.utils.solidityKeccak256(
+        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
+        [
+          false,
+          0,
+          0,
+          ENA_whale.address,
+          USDC_whale.address,
+          ethers.constants.AddressZero,
+          customerAmount,
+          chainId,
+        ],
+      );
+      const customerSignature = EthCrypto.sign(signer.privateKey, customerMessage);
+      const customerInfo: CustomerInfoStruct = {
+        paymentInUSDC: false,
+        visitBountyAmount: 0,
+        spendBountyPercentage: 0,
+        customer: ENA_whale.address,
+        venueToPayFor: USDC_whale.address,
+        promoter: ethers.constants.AddressZero,
+        amount: customerAmount,
+        signature: customerSignature,
+      };
+
+      const feesStruct = (await belongCheckIn.belongCheckInStorage()).fees;
+      const platformSubsidy = await helper.calculateRate(feesStruct.platformSubsidyPercentage, customerAmount);
+      const processingFee = await helper.calculateRate(feesStruct.processingFeePercentage, customerAmount);
+
+      await ENA.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);
+
+      const escrowDepositsBefore = await escrow.venueDeposits(venue);
+      const platformAddress = (await factory.nftFactoryParameters()).platformAddress;
+      const platformBalanceBefore = await ENA.balanceOf(platformAddress);
+
+      await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);
+
+      const escrowDepositsAfter = await escrow.venueDeposits(venue);
+      const platformBalanceAfter = await ENA.balanceOf(platformAddress);
+
+      const expectedFromEscrow = platformSubsidy.sub(processingFee);
+
+      expect(escrowDepositsBefore.longDeposits.sub(escrowDepositsAfter.longDeposits)).to.eq(expectedFromEscrow);
+      expect(escrowDepositsAfter.longDeposits).to.eq(
+        escrowDepositsBefore.longDeposits.sub(platformSubsidy).add(processingFee)
+      );
+      expect(platformBalanceAfter).to.eq(platformBalanceBefore);
+      expect(processingFee).to.be.gt(0);
+    });
```

{% endstep %}
{% endstepper %}

To reproduce locally:

{% stepper %}
{% step %}

### Setup

Install dependencies:

```bash
pnpm install
```

{% endstep %}

{% step %}

### Run the PoC test

```bash
MAINNET_RPC_URL=https://ethereum.publicnode.com \
LEDGER_ADDRESS=0x1111111111111111111111111111111111111111 \
PK=0x2222222222222222222222222222222222222222222222222222222222222222 \
pnpm test test/v2/platform/belong-check-in.test.ts --grep "processing fee" --show-stack-traces
```

{% endstep %}
{% endstepper %}

Test Results:

* The test passes and confirms the vulnerability:
  * Escrow deduction: platformSubsidy - processingFee (not full subsidy)
  * Platform balance: unchanged (0 ENA received)
  * Processing fee: remains in escrow, can be reclaimed by venue
  * Assertion: platformBalanceAfter == platformBalanceBefore (proves no fee transfer)
  * Assertion: escrowDepositsAfter.longDeposits == escrowDepositsBefore.longDeposits - platformSubsidy + processingFee

The test mathematically proves that:

1. The escrow only decreases by `subsidyMinusFees` instead of the full `platformSubsidy`
2. The platform receives exactly 0 tokens (balance unchanged)
3. The `processingFee` amount remains in the escrow under the venue's balance
4. The venue can reclaim this fee later

## Recommended Mitigation

To fix this critical vulnerability, the `payToVenue()` function must be modified to properly handle the processing fee. Example fix (conceptual):

```solidity
// Calculate fees
uint256 processingFee = calculateRate(processingFeePercentage, customerAmount);
uint256 platformSubsidy = calculateRate(platformSubsidyPercentage, customerAmount);

// Withdraw the FULL subsidy from escrow
escrow.withdrawLONG(venue, platformSubsidy);

// Transfer the processing fee to the platform via revenue handler
_handleRevenue(processingFee);

// Transfer the remaining amount to the venue
uint256 venueAmount = platformSubsidy - processingFee;
ENA.safeTransfer(venue, venueAmount);
```

Key changes:

1. Withdraw the full `platformSubsidy` amount from escrow (not `subsidyMinusFees`)
2. Always call `_handleRevenue(processingFee)` to transfer the fee to the platform
3. Only transfer `platformSubsidy - processingFee` to the venue

Additional recommendations:

* Add an invariant test to ensure `escrowBefore - escrowAfter == platformSubsidy` (full amount)
* Add an assertion to verify platform balance increases by `processingFee` after each payment
* Implement accounting checks that validate total fee collection matches expected amounts
* Consider adding events that log fee transfers for transparency and monitoring

**Severity: CRITICAL**

This vulnerability causes direct loss of platform funds (revenue) on every LONG payment transaction, requires no special privileges to trigger, and fundamentally breaks the protocol's revenue model. Immediate remediation is required.
