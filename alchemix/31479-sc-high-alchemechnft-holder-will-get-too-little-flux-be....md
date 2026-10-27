# 31479 - \[SC - High] alchemechNFT holder will get too little FLUX be...

Submitted on May 20th 2024 at 05:06:46 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31479

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

When `alchemicNFT` holders claim FLUX, the amount to claim is calculated relative to the amount that Patron NFT holders claim, which dramatically underestimates the amount to claim.

## Vulnerability Details

When NFT holders claim FLUX, the calculated amount should either by multiplied by 0.4% (for Patron NFT holders) or 0.05% (for alchemicNFT holders).

However, in the implementation of `getClaimableFlux()`, we first multiply by 0.4% to get the Patron NFT holder amount. Then, if it's an Alchemic NFT holder, we multiply that resulting value by 0.05% to get the final amount.

The result is that Alchemic NFT holders receive 0.4% \* 0.05% = 0.0002% of the calculated amount, which is 250x less than they should.

## Impact Details

Alchemic NFT holders will receive 250x less FLUX than they should when claiming. This loss of FLUX amount is permanent and can't be fixed.

## References

`FluxToken.sol`

## Proof of Concept

The following test can be added to `FluxToken.t.sol`. It should return 0.05% of `500_000`, which would equal `250`, but instead returns `1`.

```solidity
function test_InflatedAlchemicNFTClaim() external {
    uint256 amount = 500_000;
    uint256 bptCalculation = flux.getClaimableFlux(amount, alchemechNFT);
    assertEq(bptCalculation, 1);
}
```
