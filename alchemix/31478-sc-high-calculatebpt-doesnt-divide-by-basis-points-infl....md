# 31478 - \[SC - High] calculateBPT doesnt divide by basis points infl...

Submitted on May 20th 2024 at 05:03:00 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31478

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Theft of unclaimed royalties

## Description

## Brief/Intro

The `calculateBPT()` function in `FluxToken.sol` (which is used when claiming FLUX for NFT holders) inflates the result by 10000x, as it doesn't divide by basis points.

## Vulnerability Details

`bptMultiplier` sets the ratio of FLUX that patron NFT holders receive. It is intended to set the value to `0.4%` by setting it to `40` and dividing by `BPS`.

```solidity
/// @notice The ratio of FLUX patron NFT holders receive (.4%)
uint256 public bptMultiplier = 40;
```

However, when BPT is calculated, we never divide by BPS:

```solidity
function calculateBPT(uint256 _amount) public view returns (uint256 bptOut) {
    bptOut = _amount * bptMultiplier;
}
```

## Impact Details

This function is used when calculating the amount of FLUX that is claimable for NFT holders. The result is that this value will be inflated by 10000x, so 10000x more FLUX will be claimed than should be. This excess FLUX can be used for boosting bribe payments in an unfair manner (since the user should not have as much boosting ability as they receive).

## References

`FluxToken.sol`

## Proof of Concept

The following test can be added to `FluxToken.t.sol`. It should return 0.4% of amount, which would equal `40`, but instead returns `400_000`.

```solidity
function test_InflatedBPT() external {
    uint256 amount = 10_000;
    uint256 bptCalculation = flux.calculateBPT(amount);
    assertEq(bptCalculation, 400_000);
}
```
