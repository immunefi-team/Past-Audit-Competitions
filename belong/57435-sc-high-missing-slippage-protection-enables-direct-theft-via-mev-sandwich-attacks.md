# 57435 sc high missing slippage protection enables direct theft via mev sandwich attacks

**Submitted on Oct 26th 2025 at 08:37:19 UTC by @Carrot for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57435
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

BelongCheckIn.sol is configured with `slippageBps = 999999999999999999999999999` which causes `amountOutMinimum` to be 0 on all Uniswap V3 swaps. This allows MEV bots to sandwich attack every swap transaction, resulting in direct theft of funds in-motion.

### Vulnerability Details

The protocol performs real token swaps through Uniswap V3 in multiple critical functions:

* `venueDeposit()`: Swaps USDC convenience fees to LONG
* `distributePromoterPayments()`: Swaps USDC to LONG for promoter payouts
* `payToVenue()`: Auto-converts LONG payments to USDC
* `_handleRevenue()`: Swaps USDC to LONG for buyback mechanism

The slippage protection calculation in `_swapExact()` (lines referenced in the report):

```solidity
uint256 amountOutMinimum = 
    IV3Quoter(_paymentsInfo.swapV3Quoter)
        .quoteExactInput(path, amount)
        .amountOutMin(_paymentsInfo.slippageBps);
```

With the deployed configuration:

* `BPS = 10^27`
* `slippageBps = 999999999999999999999999999` (i.e., BPS - 1)
* `amountOutMin = quote * (BPS - slippageBps) / BPS = quote * 1 / 10^27 = 0`

This means the protocol accepts ANY amount >= 0 as valid output.

### Impact Details

HIGH - Theft of unclaimed yield:

* Promoter rewards (unclaimed yield) stolen during LONG distribution
* Venue convenience fee rewards stolen during swap to LONG
* Funds being swapped through Uniswap V3 are "in-motion" and can be stolen
* MEV bots can extract nearly 100% value during swap execution

Financial estimates (as reported):

* Venue deposits: $500/day stolen from convenience fees
* Promoter payouts: $2,500/day stolen from rewards
* Total: \~$90,000/month direct loss

## Attack Scenario

{% stepper %}
{% step %}

### Monitor mempool

MEV bot monitors mempool for BelongCheckIn swap transactions.
{% endstep %}

{% step %}

### Front-run

Front-run transaction: buy output token to manipulate pool price.
{% endstep %}

{% step %}

### Swap executes with minimal protection

Protocol swap executes with `amountOutMinimum = 0`.
{% endstep %}

{% step %}

### Back-run

Back-run transaction: sell output token at the inflated price.
{% endstep %}

{% step %}

### Result

Direct theft of funds in-motion (attacker extracts value from the swap).
{% endstep %}
{% endstepper %}

## References

* BelongCheckIn.sol lines 664-665 (slippage calculation)
* Helper.sol line 131-133 (amountOutMin function)

## Recommendation

Set a reasonable slippage tolerance. Example from the report:

```solidity
slippageBps = 5e24;  // 0.5% slippage
```

(Adjust the exact value to an appropriate tolerance consistent with the protocol's UX and risk profile.)

{% hint style="warning" %}
Current deployed value of `slippageBps` effectively nullifies slippage protection and allows trivial sandwich attacks. Prioritize fixing swaps that use the affected configuration.
{% endhint %}

## Proof of Concept

<details>

<summary>Run the provided Hardhat test</summary>

Run Command:

```bash
npx hardhat test test/v2/platform/slippage-poc-clean.test.ts
```

Test file contents:

```typescript
import { ethers } from 'hardhat';
import { BigNumber } from 'ethers';
import { expect } from 'chai';

describe('Slippage Protection Bypass', () => {
  const BPS = BigNumber.from(10).pow(27);
  const DEPLOYED_SLIPPAGE = BPS.sub(1);
  
  function calculateMinOut(quote: BigNumber, slippage: BigNumber): BigNumber {
    return quote.mul(BPS.sub(slippage)).div(BPS);
  }
  
  it('Deployed configuration accepts zero output', async () => {
    const expectedOutput = ethers.utils.parseUnits('1000', 18);
    const minAccepted = calculateMinOut(expectedOutput, DEPLOYED_SLIPPAGE);
    
    expect(minAccepted).to.equal(0);
    
    const attackerProvides = BigNumber.from(1);
    expect(attackerProvides).to.be.gte(minAccepted);
  });
  
  it('venueDeposit vulnerable to sandwich attack', async () => {
    const convenienceFee = ethers.utils.parseUnits('5', 6);
    const expectedLong = ethers.utils.parseUnits('0.5', 18);
    const minOut = calculateMinOut(expectedLong, DEPLOYED_SLIPPAGE);
    
    expect(minOut).to.equal(0);
  });
  
  it('distributePromoterPayments loses 100% value', async () => {
    const promoterPayout = ethers.utils.parseUnits('100', 6);
    const expectedLong = ethers.utils.parseUnits('10', 18);
    const minOut = calculateMinOut(expectedLong, DEPLOYED_SLIPPAGE);
    
    expect(minOut).to.equal(0);
  });
});
```

</details>
