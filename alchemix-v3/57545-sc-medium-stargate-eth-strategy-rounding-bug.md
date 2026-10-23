# 57545 sc medium stargate eth strategy rounding bug

**Submitted on Oct 27th 2025 at 05:58:51 UTC by @LogicalJosselyn798 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57545
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The Stargate ETH strategy (`src/strategies/optimism/StargateEthPoolStrategy.sol`) rounds deposits down to the nearest `1e12` wei but still reports the full requested amount as allocated. During withdrawal, the strategy insists that the original amount of WETH is present. Because only the rounded “truncated” amount was deposited (and the dust remains as raw ETH), any withdrawal for a non-1e12-aligned amount reverts. A single misaligned allocation permanently bricks the strategy, freezing all funds assigned to it.

## Impact Details

* **Impact type**: Permanent freezing of funds.
* **Loss surface**: Once a misaligned amount is allocated, neither the vault nor any user can withdraw the associated funds; the strategy becomes unusable.
* **Severity**: High; assets are locked forever with a single transaction.

## References

* Code: `src/strategies/optimism/StargateEthPoolStrategy.sol`
* PoC: `src/test/strategies/StargateEthPoolStrategy.t.sol` (`testDeallocateRevertsDueToRoundingDust`)
* Run: `forge test --match-test testDeallocateRevertsDueToRoundingDust`

## Link to Proof of Concept

<https://gist.github.com/therzv/0b38dfe2f0da270c1c191fcc819531b5>

## Proof of Concept

## Proof of Concept

The exploit is reproduced by `src/test/strategies/StargateEthPoolStrategy.t.sol`:

```bash
forge test --match-test testDeallocateRevertsDueToRoundingDust
```

Test outline:

1. Mint `1 ether + 123 wei` of mock WETH to the strategy.
2. Call `_allocate`; only `1 ether` is deposited after rounding.
3. Attempt `_deallocate` for the original amount.
4. The call reverts with `Strategy balance is less than the amount needed`, proving the strategy is bricked.

PoC Gist: <https://gist.github.com/therzv/0b38dfe2f0da270c1c191fcc819531b5>
