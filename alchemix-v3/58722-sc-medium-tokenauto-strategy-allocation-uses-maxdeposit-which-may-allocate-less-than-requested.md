# 58722 sc medium tokenauto strategy allocation uses maxdeposit which may allocate less than requested leaving any excess funds permanently locked

**Submitted on Nov 4th 2025 at 09:24:14 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58722
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The TokenAuto strategies in scope (TokeAutoUSDStrategy and TokeAutoEthStrategy) both use the tokenAuto router depositMax function to deposit the allocation amount. TokenAuto's depositMax function tries to deposit as much of the caller's underlying balance available, but capps the deposited amount to whatever limitation the vault imposes on the depositor in maxDeposit:

```solidity
//From the Auto Token router contract
function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        uint256 assetBalance = asset.balanceOf(msg.sender);
        uint256 maxDeposit = vault.maxDeposit(to);
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
        pullToken(asset, amount, address(this));

        approve(IERC20(vault.asset()), address(vault), amount);
        return deposit(vault, to, amount, minSharesOut);
    }
```

The Token Auto vault maxDeposit function calls MaxMint which may limit the mint amount under certain conditions, without reverting (for example to 0 if the vault is paused)

```solidity
//From Token Auto Autopool4626::maxMint

// If we are temporarily paused, or in full shutdown mode,
// no new shares are able to be minted
if (paused || $.shutdown) {
    return 0;
}
```

## Vulnerability Details

The issue is that the Alchemix TokenAuto strategies do not check that the amount minted is equal to the amount requested. Since TokenAuto might mint less (without reverting) the Myt vault allocation will succeed but only part of the funds (USDC or Weth) sent to the strategy will be deposited. The rest will remain as unallocated balance of the strategy.

Note that both TokenAuto strategies' \_deallocate function start with a check of the difference between the shares needed to cover the deallocation and the shares held by the strategy:

```solidity
//From TokeAutoUSDStrategy

function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoUSD.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        // account for vault rounding up
```

This code assumes that the shares held by the strategy are equal or more than the shares needed for the deallocation (otherwise the line `uint256 shareDiff = actualSharesHeld - sharesNeeded;` reverts with underflow). However, in the scenario of a partial allocation described above, the strategy will hold less shared than requested. This means that in the case of a partial allocation, any attempt to deallocate the amounts not deposited will fail. This leaves any such amounts permanently locked within the strategy.

## Impact Details

Permanent lock of any vault funds allocated to Token Auto strategies where the actual allocation is partial.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoUSDStrategy.sol#L42>

## Proof of Concept

## Proof of Concept

How to run:

1. Copy the code below into the TokeAutoUSDStrategyTest contract in /v3-poc/src/test/strategies/TokeAutoUSDStrategy.t.sol
2. add the IERC4626 import as bellow and change the BaseStrategyTest import to import the entire file:

```solidity
import {IERC4626} from "lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
import "../libraries/BaseStrategyTest.sol";
```

3. run with `FOUNDRY_PROFILE=default forge test --fork-url https://mainnet.gateway.tenderly.co --match-test testPartialAllocation -vvv`

```solidity
function testPartialAllocation() public {
        
    uint256 initialDeposit = getTestConfig().vaultInitialDeposit;
    uint256 amountFirstAllocation = initialDeposit / 2;

    //Emulate TokenAuto vault maxDeposit limitation to half the requested amount
    vm.mockCall(
        address(TOKE_AUTO_USD_VAULT),
        abi.encodeWithSelector(IERC4626.maxDeposit.selector, strategy),
        abi.encode(amountFirstAllocation / 2)
    );
    
    //Trying to allocate 500 USDC
    vm.startPrank(allocator);
    uint256 oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
    bytes memory dataOldAlloc = abi.encode(oldAllocation);
    VaultV2(vault).allocate(strategy, dataOldAlloc, amountFirstAllocation);
    vm.stopPrank();

    uint256 realAssets = IMYTStrategy(strategy).realAssets();
    uint256 freeBalance = IERC20(USDC).balanceOf(strategy);
    console.log("strategy state after allocation: realAssets(): %s. USDC balance: %s",realAssets/1e6, freeBalance / 1e6);
    /*  Output:
        strategy state after allocation: realAssets(): 250. USDC balance: 250

        Only half of the requested amount deposited
    */


    //Dealocating the part that was deposited succeeds
    vm.startPrank(allocator);
    oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
    dataOldAlloc = abi.encode(oldAllocation);
    VaultV2(vault).deallocate(strategy, dataOldAlloc, realAssets);
    vm.stopPrank();

    realAssets = IMYTStrategy(strategy).realAssets();
    freeBalance = IERC20(USDC).balanceOf(strategy);
    console.log("strategy state after deallocation of its deposited funds: realAssets(): %s. USDC balance: %s",realAssets/1e6, freeBalance / 1e6);

    //Try to dealocate the non-deposited funds fails with an underflow
    vm.startPrank(allocator);
    oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
    dataOldAlloc = abi.encode(oldAllocation);
    vm.expectRevert();
    VaultV2(vault).deallocate(strategy, dataOldAlloc, freeBalance);
    vm.stopPrank();

}
```
