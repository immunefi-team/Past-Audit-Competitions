# 56336 sc insight stargateethpoolstrategy deallocate would emit false deallocating loss event in some cases

**Submitted on Oct 14th 2025 at 17:17:36 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56336
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**

## Description

## Brief/Intro

the `_deallocate` amount param is capped to `lpBalance` if the amount param is greater than the contract balance. but the check `ethRedeemed < amount` is using amount for comparison which is not always the amount used for redeem because the above cap mechanism.

## Vulnerability Details

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        // Compute LP needed ∝ TVL to withdraw `amount` underlying
        // For Stargate, LP tokens are 1:1 with underlying
        // So we can just redeem the amount directly
        uint256 lpBalance = lp.balanceOf(address(this));
        uint256 lpNeeded = amount; // 1:1 ratio

        // Cap at available LP balance
        if (lpNeeded > lpBalance) {
@>          lpNeeded = lpBalance;
        }

        // Redeem LP to native ETH, then wrap back to WETH
        lp.approve(address(pool), lpNeeded);
        uint256 ethBalanceBefore = address(this).balance;
@>      pool.redeem(lpNeeded, address(this));
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
        // @audit
        if (ethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss which includes rounding loss.", amount, ethRedeemed);
        }

```

here we can see that if the `amount` assigned to `lpNeeded`. for some case where it is capped to `lpBalance`, the redeem call would use this `lpBalance` that always result in lower `ethRedeemed` than the `amount` thus it would always emitting the event which is not necessarily true.

## Impact Details

possibly emitting deallocation loss when it is not

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/strategies/optimism/StargateEthPoolStrategy.sol#L73-L74>

## Proof of Concept

## Proof of Concept

1. contract deallocating 100 LP amount
2. currently the contract is having 99.8 LP amount
3. cap the amount, and call `pool.redeem` using 99.8 LP, and get 99.8 ether
4. check if the `ethRedeemed < amount` which is `99.8 < 100`
5. emit the deallocation loss event, even though that is all LP token the contract have and no deallocation loss happen
