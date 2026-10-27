# 57583 sc low promoter bounty bait and switch via updatevenuerules

**Submitted on Oct 27th 2025 at 10:46:11 UTC by @jo13 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57583
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

Venues decide how much they reward promoters via the `bountyType` field inside `VenueRules`. A dishonest venue can lure promoters by first setting the most generous option (`Both`) and later—right before customers start paying—downgrade the bounty to a cheaper type or disable it altogether. Because customer payloads are signed without any link to a specific rules version, `payToVenue` simply reads whatever bounty rules are active when the transaction lands. The venue still receives its money, but the promoter ends up with fewer (or zero) credits.

In short, a venue can pull a bait-and-switch on promoter rewards without breaking its own revenue flow.

## Vulnerability Details

The loophole is a combination of two design choices:

* Signed payloads ignore rule versions\
  Customer signatures do not include a `rulesVersion` or `rulesHash`. At execution, `checkCustomerInfo` compares the computed `bountyType` against the venue’s current rules, not the rules under which the payload was signed.
* Venue can change rules at any time

```solidity
// contracts/v2/platform/BelongCheckIn.sol
function updateVenueRules(VenueRules calldata rules) external {
    uint256 venueId = msg.sender.getVenueId();
    uint256 venueBalance = belongCheckInStorage.contracts.venueToken.balanceOf(msg.sender, venueId);
    require(venueBalance > 0, NotAVenue());
    _setVenueRules(msg.sender, rules);
}

function _setVenueRules(address venue, VenueRules memory rules) private {
    require(rules.paymentType != PaymentTypes.NoType, WrongPaymentTypeProvided());
    generalVenueInfo[venue].rules = rules;
    emit VenueRulesSet(venue, rules);
}
```

* `payToVenue` enforces the current rules

```solidity
// contracts/v2/utils/SignatureVerifier.sol
function checkCustomerInfo(address signer, CustomerInfo calldata customerInfo, VenueRules memory rules) external view {
    PaymentTypes paymentType = customerInfo.paymentInUSDC ? PaymentTypes.USDC : PaymentTypes.LONG;
    require(
        rules.paymentType != PaymentTypes.NoType &&
        (rules.paymentType == PaymentTypes.Both || rules.paymentType == paymentType),
        WrongPaymentType()
    );

    if (customerInfo.promoter != address(0)) {
        BountyTypes bountyType = customerInfo.visitBountyAmount > 0 && customerInfo.spendBountyPercentage > 0
            ? BountyTypes.Both
            : customerInfo.visitBountyAmount > 0 && customerInfo.spendBountyPercentage == 0
                ? BountyTypes.VisitBounty
                : customerInfo.visitBountyAmount == 0 && customerInfo.spendBountyPercentage > 0
                    ? BountyTypes.SpendBounty
                    : BountyTypes.NoType;
        require(rules.bountyType == bountyType && bountyType != BountyTypes.NoType, WrongBountyType());
    }

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

Note the signature omits any rules binding; `rules` are passed separately and compared at call time.

### Attack path: bait-and-switch

{% stepper %}
{% step %}

### Attract

Venue sets `bountyType = Both` (or a higher-paying type) to attract promoters and traffic.
{% endstep %}

{% step %}

### Lower terms

Before (or during) customer payments, venue updates rules to a lower-paying type (`VisitBounty`, `SpendBounty`) or `NoType`.
{% endstep %}

{% step %}

### Execution

Platform signs new `CustomerInfo` payloads that match the current rules; `payToVenue` mints promoter credits per the lowered type (or none if `NoType`). No revert is needed; venue still gets paid.
{% endstep %}

{% step %}

### Payout

`distributePromoterPayments` pays out based on promoter credits. Since fewer credits were minted, the promoter receives less than the originally advertised amount.
{% endstep %}
{% endstepper %}

## Impact

* Lost earnings for promoters – Rewards shrink or disappear for visits they rightfully drove.
* Erosion of trust – Promoters and customers cannot rely on published bounty terms.
* Minimal risk to venue – The venue’s own revenue is unaffected, so the incentive to cheat is high.

Severity: High (economic fairness + reputational damage).

## References

* Code paths
  * `contracts/v2/platform/BelongCheckIn.sol`: `updateVenueRules`, `_setVenueRules`, `payToVenue`
  * `contracts/v2/utils/SignatureVerifier.sol`: `checkCustomerInfo`
* Data structures
  * `contracts/v2/Structures.sol`: `VenueRules`, `CustomerInfo`, `BountyTypes`, `PaymentTypes`

## Proof of Concept

Add this test to `/home/jo/audit-comp-belong/test/v2/platform/belong-check-in-bsc-fork.test.ts` and run:

LEDGER\_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x1000000000000000000000000000000000000000000000000000000000000000001 npx hardhat test --grep "promoter rules bait-and-switch" test/v2/platform/belong-check-in.test.ts

```typescript
describe('Security: promoter rules bait-and-switch', () => {
    it('stale Both payload reverts after venue lowers bountyType to Visit', async () => {
      const {
        belongCheckIn,
        helper,
        signatureVerifier,
        signer,
        referral,
        USDC,
        USDC_whale,
        ENA_whale,
      } = await loadFixture(fixture);

      // 1) Venue deposits with generous rules (Both) to gain credits and set initial rules
      const uri = 'rules:both';
      const venueAmount = await u(100, USDC);
      const venue = USDC_whale.address;
      const venueMsg = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const venueSig = EthCrypto.sign(signer.privateKey, venueMsg);
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 3, bountyType: 3, longPaymentType: 0 } as VenueRulesStruct, // Both
        venue,
        amount: venueAmount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature: venueSig,
      };
      const willBeTaken = convenienceFeeAmount.add(venueAmount);
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      // 2) Prepare a valid CustomerInfo for Both (visit>0 AND spend%>0)
      const customerAmount = await u(5, USDC);
      await USDC.connect(USDC_whale).transfer(ENA_whale.address, customerAmount);
      await USDC.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);
      const customerMsg = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [true, await u(1, USDC), 1000, ENA_whale.address, venue, referral.address, customerAmount, chainId],
      );
      const customerSig = EthCrypto.sign(signer.privateKey, customerMsg);
      const customerInfoBoth: CustomerInfoStruct = {
        paymentInUSDC: true,
        visitBountyAmount: await u(1, USDC),
        spendBountyPercentage: 1000,
        customer: ENA_whale.address,
        venueToPayFor: venue,
        promoter: referral.address,
        amount: customerAmount,
        signature: customerSig,
      };
      console.log('[bait-switch] derived bounty from payload:', {
        visitBountyAmount: customerInfoBoth.visitBountyAmount.toString(),
        spendBountyPercentage: customerInfoBoth.spendBountyPercentage.toString(),
        derivedBountyType: 'Both',
      });

      // 3) Venue lowers rules to Visit-only (bountyType = VisitBounty)
      await belongCheckIn
        .connect(USDC_whale)
        .updateVenueRules({ paymentType: 3, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct);
      const rulesAfter = await belongCheckIn.generalVenueInfo(venue);
      console.log('[bait-switch] rules updated before execution:', {
        paymentType: rulesAfter.rules.paymentType.toString(),
        bountyType: rulesAfter.rules.bountyType.toString(),
        longPaymentType: rulesAfter.rules.longPaymentType.toString(),
      });

      // 4) Using the stale Both payload must now revert with WrongBountyType (rules are checked at call time)
      console.log('[bait-switch] attempting payToVenue with stale Both payload; expecting WrongBountyType revert');
      await expect(belongCheckIn.connect(ENA_whale).payToVenue(customerInfoBoth)).to.be.revertedWithCustomError(
        signatureVerifier,
        'WrongBountyType',
      );
      console.log('[bait-switch] got expected WrongBountyType revert — venue reduced promoter bounty vs payload');
    });

    it('after lowering to Visit, a new Visit-only payload mints only visit credits', async () => {
      const {
        belongCheckIn,
        helper,
        signatureVerifier,
        signer,
        referral,
        promoterToken,
        USDC,
        USDC_whale,
        ENA_whale,
      } = await loadFixture(fixture);

      // 1) Venue deposits with Both, then lowers to Visit-only
      const uri = 'rules:visit-only';
      const venueAmount = await u(100, USDC);
      const venue = USDC_whale.address;
      const venueMsg = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const venueSig = EthCrypto.sign(signer.privateKey, venueMsg);
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 3, bountyType: 3, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature: venueSig,
      };
      const willBeTaken = convenienceFeeAmount.add(venueAmount);
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);
      await belongCheckIn
        .connect(USDC_whale)
        .updateVenueRules({ paymentType: 3, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct);

      // 2) Prepare a Visit-only CustomerInfo (visit>0, spend%==0) consistent with lowered rules
      const visitOnly = await u(1, USDC);
      const customerAmount = await u(5, USDC);
      await USDC.connect(USDC_whale).transfer(ENA_whale.address, customerAmount);
      await USDC.connect(ENA_whale).approve(belongCheckIn.address, customerAmount);
      const customerMsgVisit = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [true, visitOnly, 0, ENA_whale.address, venue, referral.address, customerAmount, chainId],
      );
      const customerSigVisit = EthCrypto.sign(signer.privateKey, customerMsgVisit);
      const customerInfoVisit: CustomerInfoStruct = {
        paymentInUSDC: true,
        visitBountyAmount: visitOnly,
        spendBountyPercentage: 0,
        customer: ENA_whale.address,
        venueToPayFor: venue,
        promoter: referral.address,
        amount: customerAmount,
        signature: customerSigVisit,
      };

      const venueId = await helper.getVenueId(venue);
      const promoterBalBefore = await promoterToken.balanceOf(referral.address, venueId);

      const tx = await belongCheckIn.connect(ENA_whale).payToVenue(customerInfoVisit);

      const promoterBalAfter = await promoterToken.balanceOf(referral.address, venueId);
      await expect(tx).to.emit(belongCheckIn, 'CustomerPaid');
      // Only the visit bounty should be minted when rules are Visit-only
      expect(promoterBalAfter.sub(promoterBalBefore)).to.eq(visitOnly);
    });
  });
```
