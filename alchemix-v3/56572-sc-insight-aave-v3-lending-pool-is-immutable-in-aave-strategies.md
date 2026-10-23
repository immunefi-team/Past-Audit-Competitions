# 56572 sc insight aave v3 lending pool is immutable in aave strategies

**Submitted on Oct 17th 2025 at 19:51:02 UTC by @kodyvim for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56572
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Both `AaveV3ARBUSDCStrategy` and `AaveV3ARBWETHStrategy` contract hardcodes the AAVE V3 lendingPool address as an immutable variable.

## Vulnerability Details

This violates AAVE’s integration guidelines, which recommend dynamically querying the PoolAddressProvider for the current lendingPool address. If AAVE updates or migrates the pool, the contract will point to a deprecated address, disrupting some core logic of the protocol (for example: rendering it unable to deposit, withdraw, or access funds in the new pool).

```solidity
IERC20 public immutable usdc; // ARB USDC
   @> IAavePool public immutable pool; // Aave v3 Pool on ARB
    IAaveAToken public immutable aUSDC; // aToken for USDC on ARB

    constructor(address _myt, StrategyParams memory _params, address _usdc, address _aUSDC, address _pool, address _permit2Address)
        MYTStrategy(_myt, _params, _permit2Address, _usdc)
    {
        usdc = IERC20(_usdc);
      @>  pool = IAavePool(_pool);
        aUSDC = IAaveAToken(_aUSDC);
    }
```

## Impact Details

If AAVE migrates to a new lendingPool address:

Loss of functionality: the AaveV3Farm contract will continue to point to the old, deprecated pool address, which will no longer hold funds or support operations. As a result `supply` will fail or send funds to an empty pool and withdraw will fail, locking users out of their funds.

## Recommendation

Update `AaveV3ARBUSDCStrategy` and `AaveV3ARBWETHStrategy` contract to dynamically fetch the `pool` address from the `PoolAddressProvider` before each AAVE interaction:

Replace address public immutable `pool` with a reference to the `PoolAddressProvider`.

Store the PoolAddressProvider address (which is stable and does not change) in the constructor. Before each AAVE operation (supply, withdraw, etc.), call IAddressProvider(poolAddressProvider).getPool to get the current `pool` address.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol?utm\\_source=immunefi#L27> <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol#L27>

## Proof of Concept

## Proof of Concept

`Deployment:` The `AaveV3ARBUSDCStrategy` contract is deployed with lendingPool set to the current AAVE V3 lendingPool address. allocate now successfully depositing assets into AAVE.

`AAVE Migration:` AAVE’s governance migrates to a new lendingPool address (e.g., 0xNewLendingPool) and moves all funds (liquidity, reserves) to the new contract. The PoolAddressProvider::getPool function now returns a NewLendingPool, and OldLendingPool is deprecated (no funds, no operations).

`Contract Failure:` The contract attempts IAaveV3Pool(OldLendingPool).supply(...), which fails or sends funds to the deprecated pool, where they become stuck. calls withdraw via `deallocate` to retrieve funds. The contract calls IAaveV3Pool(0xOldLendingPool).withdraw(...), which fails because the old pool has no funds.
