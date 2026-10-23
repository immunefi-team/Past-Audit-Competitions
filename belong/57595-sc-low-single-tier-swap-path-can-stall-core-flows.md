# 57595 sc low single tier swap path can stall core flows

**Submitted on Oct 27th 2025 at 11:50:21 UTC by @koko7 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57595
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

`_buildPath` in `BelongCheckIn` is hard-wired to one Uniswap-V3 fee tier. If liquidity migrates to another tier the function reverts with `NoValidSwapPath`, and every call that needs a swap fails. Deposits, promoter payouts and fee buy-backs all depend on these swaps, so a missing pool turns into a full protocol outage. The entire transaction reverts, so funds are safe, but the platform is effectively frozen until the fee tier is updated.

### Vulnerability Details

The path builder only checks a single configured fee tier and reverts if no pool exists at that fee. Example from the code:

```solidity
// contracts/v2/platform/BelongCheckIn.sol::_buildPath
function _buildPath(
    PaymentsInfo memory _paymentsInfo,
    address tokenIn,
    address tokenOut
) internal view returns (bytes memory path) {
    // Direct pool
    if (
        IV3Factory(_paymentsInfo.swapV3Factory).getPool(
            tokenIn,
            tokenOut,
            _paymentsInfo.swapPoolFees
        ) != address(0)
    ) {
        path = abi.encodePacked(
            tokenIn,
            _paymentsInfo.swapPoolFees,
            tokenOut
        );
    }
    // tokenIn -> W_NATIVE_CURRENCY -> tokenOut
    else if (
        IV3Factory(_paymentsInfo.swapV3Factory).getPool(
            tokenIn,
            _paymentsInfo.wNativeCurrency,
            _paymentsInfo.swapPoolFees
        ) != address(0)
    ) {
       // @note here 
        path = abi.encodePacked(
            tokenIn,
            _paymentsInfo.swapPoolFees,//@ same swap fees 
            _paymentsInfo.wNativeCurrency,
            _paymentsInfo.swapPoolFees,//@ same swap fees 
            tokenOut
        );
    } else {
        revert NoValidSwapPath();
    }
}
```

Key points:

* Only checks one configured fee tier (`swapPoolFees`).
* No probing of alternative tiers (500, 3000, 10000) or per-leg fee combinations.
* All swaps flow through this builder via `_swapExact` → Quoter → Router; callers include `venueDeposit`, `distributePromoterPayments`, `payToVenue` (AutoConvert), and `_handleRevenue`.

## Impact

When the chosen fee tier has no pool, every USDC↔LONG swap reverts and the protocol grinds to a halt.

What breaks:

* Venue deposits (USDC → LONG for convenience/affiliate fees and buy-back).
* Promoter payouts (USDC → LONG fee burn; LONG payouts that start in USDC).
* Customer payments that auto-convert LONG → USDC.

Business effect: Operations pause until the owner updates the fee tier. Funds remain safe, but deposits, payouts, and burns are blocked—an outage, not a loss.

Severity: High.

## References

* Code: `contracts/v2/platform/BelongCheckIn.sol`
  * `_buildPath` (single-tier discovery with hard revert)
  * `_swapUSDCtoLONG`, `_swapLONGtoUSDC`, `_swapExact`
  * Callers: `venueDeposit`, `distributePromoterPayments`, `payToVenue`, `_handleRevenue`
* Uniswap V3 fee tiers: 500 (0.05%), 3000 (0.3%), 10000 (1.0%)

## Proof of Concept

<details>

<summary>Test that triggers NoValidSwapPath by setting a non-standard fee tier</summary>

Add this test to `/home/jo/audit-comp-belong/test/v2/platform/belong-check-in-bsc-fork.test.ts` and run:

LEDGER\_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x1000000000000000000000000000000000000000000000000000000000000001 npx hardhat test --grep "Inflexible swap path discovery" test/v2/platform/belong-check-in-bsc-fork.test.ts

```ts
describe('Security: Inflexible swap path discovery', () => {
    it('reverts with NoValidSwapPath when configured fee tier has no pools for both direct and WBNB route', async () => {
      const { belongCheckIn, admin, helper, signer, USDC, USDC_whale } = await loadFixture(fixture);

      // Set an invalid/unavailable fee tier to force _buildPath to fail
      const wrongFeeTier = 4242 as unknown as number; // non-standard tier
      const paymentsInfoNew = {
        slippageBps: paymentsInfo.slippageBps,
        swapPoolFees: wrongFeeTier,
        swapV3Factory: paymentsInfo.swapV3Factory,
        swapV3Router: paymentsInfo.swapV3Router,
        swapV3Quoter: paymentsInfo.swapV3Quoter,
        wNativeCurrency: paymentsInfo.wNativeCurrency,
        usdc: paymentsInfo.usdc,
        long: paymentsInfo.long,
        maxPriceFeedDelay: paymentsInfo.maxPriceFeedDelay,
      } as BelongCheckIn.PaymentsInfoStruct;

      await belongCheckIn.connect(admin).setParameters(paymentsInfoNew, fees, stakingRewards);

      // Prepare a valid venue deposit that would trigger USDC->CAKE swap for convenience fee
      const uri = 'no-path-expected';
      const amount = await u(10, USDC);
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

      await expect(belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo)).to.be.revertedWithCustomError(
        belongCheckIn,
        'NoValidSwapPath',
      );
    });
  });
```

</details>
