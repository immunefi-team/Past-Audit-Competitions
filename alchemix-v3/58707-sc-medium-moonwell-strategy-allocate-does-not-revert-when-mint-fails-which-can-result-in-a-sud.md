# 58707 sc medium moonwell strategy allocate does not revert when mint fails which can result in a sudden drop in myt share price and consequently sever under collateralization

**Submitted on Nov 4th 2025 at 07:32:46 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58707
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Both MoonwellUSDCStrategy and MoonwellWETHStrategy implement \_allocate() by calling the moonwell mToken mint function with the allocated funds:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
    // Mint mUSDC with underlying USDC
    mUSDC.mint(amount);
    return amount;
}
```

Moonwell MErc20's implementation of mint returns an error value instead of reverting for some error types that might occur during minting (such as the mint being rejected by the comptroller of math errors):

```solidity
//From Moonwell MErc20
 function mint(uint mintAmount) external override returns (uint) {
    (uint err, ) = mintInternal(mintAmount);
    return err;
}
```

## Vulnerability Details

Since the Moonwell strategies do not check the return value from mERC20::mint and revert when an error occurs, it might happen that the mToken does not mint any funds, returns an error, but the Strategy allocation transaction succeeds, leaving the USDC/Weth transffered in from the Myt vault as non-deposited balance of the Strategy.

A side-effect from such a scenario is that the strategy's realAssets() function will return 0 (inspite of the fact that it holds non-deposited funds), since realAssets() is based only on deposited funds:

```solidity
function realAssets() external view override returns (uint256) {
    // Use stored exchange rate and mToken balance to avoid state changes during static calls
    uint256 mTokenBalance = mUSDC.balanceOf(address(this));
    if (mTokenBalance == 0) return 0;
    uint256 exchangeRate = mUSDC.exchangeRateStored();
    // Exchange rate is scaled by 1e18, so we need to divide by 1e18
    return (mTokenBalance * exchangeRate) / 1e18;
}
```

Consequently, the Myt share price (which is calculated in convertToAssets based on the aggregated adapters' realAssets() plus the Myt vault balance) will experience a sudden drop.

Note that the USDC/Weth left in the strategy can easily be transffered back to the Myt vault by calling deallocate (which will deallocate from the strategy free balance even if it is not curretly depoloyed to Moonwell). However, while VaultV2 enables sudden drops in share price, it limits price increases through the maxRate setting (which can be set up to a maximum 200% APR), Which means it might take a very long time for the price to recuperate to it's real value.

This situation enables the following exploit path:

### attack scenario

1. Option A: An attacker who holds a privilaged Moonwell role that enables temporarily disabling mints for the Strategy, monitors for a large allocation and frontruns with a tx that disallows Strategy mints.\
   Options B: An attacker who is aware of the issue monitors for a large allocation that is expected to fail naturally (due to temporary mint disabling, expeceted math error or other reason).
2. Myth vault makes an allocation of 50% of total funds to the Moonwell vault. The allocation completes without revert even though no funds were deposited to Moonwell. As a result the Myt Vault share price immediately drops 50% (since the strategy, who now holds 50% of the Myth vault funds, report 0 realAssets).
3. The sharp drop in Myt share price is likely to cause bad debt/make many CDPs liquidatable.
4. The attacker batch liquidates as many CDPs as possible for the fees (Since AlchemistV3 is in high-ltv state, fees are maximized due to full liquidations, and are taken from the fee-vault).
5. In addition, the attacker accumulates matured redemptions ahead of time (either buying their nfts or pre-submitting themselves)and redeems them.
6. Since Myt vault share price is now artificially reduced, all redemptions benefit the redeemer on the account of CDP holders. This is because the low price is based on an underestimation of funds. The share price will inevidably crawl back up to its real value (at a rate limited by maxRate) until it represents the real totalAssets.

## Impact Details

1. Loss to CDP holders as their collateral is liquidated/redeemed for half it's real price.
2. Likely also protocol insolvancy as a result of the sharp drop in Collateral value (as precieved by the system)

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellUSDCStrategy.sol#L48>

## Proof of Concept

## Proof of Concept

How to run:

1. Copy the code below into the MoonwellUSDCStrategyTest contract in /v3-poc/src/test/strategies/MoonwellUSDCStrategy.t.sol
2. Add the following import and interface at the top of the file:

```solidity
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

interface IMoonwellERC20 {
    function mint(uint mintAmount) external virtual returns (uint);
}
```

3. Run with `FOUNDRY_PROFILE=default forge test --fork-url https://optimism.gateway.tenderly.co --match-test testMoonwellMintFailure --isolate -vvv`\
   Note: the --isolate flag is required to properly see VaultV2 price changes since VaultV2 prevents multiple price changes within a single transaction, and calls made from a forge test appear as part of a single transaction (unless --isolate is specified)

```solidity
function testMoonwellMintFailure() public {

        //Allocate amount: half of the Myt vault deposits
        uint256 allocationAmount = getTestConfig().vaultInitialDeposit / 2;

        uint256 vaultSharePriceAtStart = VaultV2(vault).convertToAssets(1e18);
        console.log("Myt vault share price at start: %s\n\n", vaultSharePriceAtStart);

        //Emulate Moonwell mERC20::mint failing due to COMPTROLLER_REJECTION
        vm.mockCall(
            address(MOONWELL_USDC_MTOKEN),
            abi.encodeWithSelector(IMoonwellERC20.mint.selector, allocationAmount),
            abi.encode(3) //COMPTROLLER_REJECTION
        );

        //Allocate allocationAmount (Result: No funds are minted and an error is returned but the allocation doesn't revert)
        vm.startPrank(allocator);
        VaultV2(vault).setMaxRate(200e16 / uint256(365 days));
        uint256 oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
        bytes memory dataOldAlloc = abi.encode(oldAllocation);
        VaultV2(vault).allocate(strategy, dataOldAlloc, allocationAmount);
        vm.stopPrank();


       uint256 realAssets = IMYTStrategy(strategy).realAssets();
        uint256 freeBalance = IERC20(USDC).balanceOf(strategy);
        console.log(
            "Strategy real assets after failed allocation: %s.\nstrategy free balance after failed allocation: %s",
            realAssets / 1e6,
            freeBalance / 1e6
        );

        //update Myt vault and check its share price after the failed allocation
        vm.warp(block.timestamp+10);
        VaultV2(vault).accrueInterest();
         uint256 vaultPriceAfter = VaultV2(vault).convertToAssets(1e18);
        console.log("Myt Vault price after failed allocation: %s\n", vaultPriceAfter);

        /*Output:
        Strategy real assets after failed allocation: 0.
        strategy free balance after failed allocation: 500
        Myt Vault price after failed allocation: 500000

        Myt Vault price fell 50%
        */
       
       
        //Deallocate the USDC stuck in the strategy
        //(Deallocate doesn't fail as it uses the free balance. The free balance is returned to the Myt Vault)
        vm.startPrank(allocator);
        oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
        dataOldAlloc = abi.encode(oldAllocation);
        VaultV2(vault).deallocate(strategy, dataOldAlloc, freeBalance);
        vm.stopPrank();

        realAssets = IMYTStrategy(strategy).realAssets();
        freeBalance = IERC20(USDC).balanceOf(strategy);
        uint256 vaultFreeBalance = IERC20(USDC).balanceOf(address(vault));
        console.log(
            "strategy real assets after Deallocation: %s.\n Strategy free balance after Deallocation: %s \n Myt Vault free balance after Deallocation: %s",
            realAssets / 1e6,
            freeBalance / 1e6,
            vaultFreeBalance / 1e6
        );
        

        //Check vault price after deallocating the funds back to the vault
        VaultV2(vault).accrueInterest();
        vaultPriceAfter = VaultV2(vault).convertToAssets(1e18);
        console.log("Vault price after Deallocation: %s\n", vaultPriceAfter);
        /*Output:
        strategy real assets after Deallocation: 0.
        Strategy free balance after Deallocation: 0 
        Myt Vault free balance after Deallocation: 1000
        Vault price after Deallocation: 500000

        after deallocation the USDC returns to the Myt Vault but the price remains 50%
        */

        //check vault price after fast forwarding 20 days
        vm.warp(block.timestamp+60*60*24*20);
        VaultV2(vault).accrueInterest();
        vaultPriceAfter = VaultV2(vault).convertToAssets(1e18);
        console.log("Vault price after warping 20 days: %s", vaultPriceAfter);

        /*Output:
        Vault price after warping 20 days: 554794

        after 20 days the vault share price recovered a bit but still far from its true value
        */

    }
```
