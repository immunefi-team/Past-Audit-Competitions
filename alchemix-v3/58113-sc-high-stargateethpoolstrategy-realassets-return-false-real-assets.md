# 58113 sc high stargateethpoolstrategy realassets return false real assets&#x20;

**Submitted on Oct 30th 2025 at 18:22:25 UTC by @silver\_eth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58113
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Introduction

The strategy in question incorrectly reports the value of its underlying Stargate LP position using:

```solidity
return pool.redeemable(address(this));
```

This function reflects **only the currently redeemable (credit-backed) liquidity**, *not* the full amount of assets actually owned by the vault. Because Stargate pools use a **credit-based redemption cap**, `redeemable()` can be significantly lower than the vault’s total LP value during periods of low cross-chain credit availability.

As a result, the vault may report **phantom losses**, misprice shares, and allow **value extraction between depositors**, ultimately causing cascading loss of user funds and protocol instability.

***

## Vulnerability Details

In Stargate, liquidity and credit are decoupled:

* `liquidity` → actual tokens deposited in the pool.
* `credit` → portion of liquidity currently available for redemption (subject to cross-chain inflows and outflows).

The function `pool.redeemable(address)` returns:

```solidity
function redeemable(address _owner) external view returns (uint256 amountLD) {
    uint256 cap = _sd2ld(paths[localEid].credit);
    ...
    return min(_ownerLPBalance, cap);
}
```

Thus, if the Stargate pool’s **available credit** is low, the returned amount is limited by `cap`, even if the vault holds far more LP tokens.

However, the strategy’s `realAssets()`uses this number as the “real” value:

```solidity
function realAssets() public view override returns (uint256) {
    return pool.redeemable(address(this));  // ❌ incorrect
}
```

### Why this is wrong

* The LP tokens held by the strategy always represent its proportional ownership of the pool’s **entire liquidity**, not just currently redeemable credit.
* `redeemable()` is a *dynamic liquidity ceiling*, not an ownership measure.
* When cross-chain credits drop (e.g., heavy outbound traffic from this chain), the function can report a fraction of actual value, misrepresenting the vault’s holdings.

Consequently, the vault’s share accounting becomes desynchronized from reality — the share price and totalAssets fluctuate purely due to Stargate’s internal credit mechanics, not real asset movement.

***

## Impact Details

### 1️⃣ Share Price Depression / Phantom Losses

During low credit periods, `totalAssets()` underreports real value. Because share price = `totalAssets / totalSupply`, it appears that shares have lost value, even though no funds were lost.

Users who withdraw during this period will redeem fewer tokens than they deposited — **real losses caused by accounting, not market movement**.

***

### 2️⃣ New Depositors Can Steal Value from Old Depositors

A malicious or lucky depositor can exploit temporary low credit to mint shares at an artificially low price.

#### Example (using real PoC numbers)

1. **Bob** deposits **1288.33 ETH** → receives **1288.33 shares**
   * `pool.redeemable()` (credit cap) = 232.08 ETH
2. **Alice** deposits 232.08 ETH. The vault computes:

   ```
   newShares = depositAmount * totalShares / totalAssets
             = 232.08 * 1288.33 / 232.08 = 1288.33 shares
   ```

   So Alice mints the same number of shares as Bob, despite depositing far less.
3. When credit later restores and total assets reflect the full **1520.41 ETH**, both Bob and Alice own equal shares → each entitled to \~760 ETH. ⇒ **Alice profits**, **Bob loses** — pure misaccounting.

***

### 3️⃣ Cascading Liquidations in Dependent Protocols

in alchemist, the artificial drop in `totalAssets` leads to:

* lowers share value,
* triggers liquidations,
* forces users to pay liquidation penalties despite no real asset loss.

***

### 4️⃣ Supply Inflation via Mispriced Deposits

If credit temporarily collapses (e.g., redeemable ≈ 100 wei), new deposits mint disproportionately large shares. When credit later normalizes, share supply is inflated, permanently diluting honest depositors and destabilizing vault accounting.

***

## References

* [Stargate V2 Docs – Credit Allocation System](https://stargateprotocol.gitbook.io/stargate/v2-developer-docs/integrate-with-stargate/credit-allocation-system)
* [`redeemable(address)` implementation (Etherscan/BscScan)](https://bscscan.com/address/0x962bd449e630b0d928f308ce63f1a21f02576057#code)
* [`PathLib.decreaseCredit()` source reference](https://arbiscan.io/address/0xe8cdf27acd73a434d661c84887215f7598e7d0d3#code)

***

## Proof of Concept

## Proof of Concept

this basic test can be placed directly in the StargatePoolStrategyTest

```
    function testRedeemableReturnsBadData() public {
        uint256 maxAvailableRedeemable = IStargatePool(STARGATE_ETH_POOL).redeemable(address(0));
        uint256 tvl = IStargatePool(STARGATE_ETH_POOL).tvl();
        console.log(maxAvailableRedeemable, tvl);
        //~232 ETH available, ~1288 ETH in tvl
    }
```

it shows that the current credit as of the fork block was less than 20% of the total tvl
