# 57311 sc medium moonwell allocation and deallocation can fail silently causing incorrect state updates and loss of yield

**Submitted on Oct 25th 2025 at 06:29:13 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57311
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Moonwell MYT strategies, namely `MoonwellUSDCStrategy` , `MoonwellWETHStrategy` on mainnet and OP, allocate funds to the underlying protocol by calling `mToken.mint()` and `mToken.redeemUnderlying()`. Those functions however can fail silently without reverting, but instead returning error codes which are not checked, causing the vault to think that it has allocated/deallocated assets from the strategy when in fact it has not. As a result, wrong `realAssets` are reported and no yield for the USDC that sits idle in the contract is generated.

## Vulnerability Details

Allocation code for Moonwell strategies looks like the following. We will use the USDC strategy for reference here.

```solidity
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
        // Mint mUSDC with underlying USDC
        // @audit, mUSDC mint can potentially revert. if this succeeds, it returns an error code of 0
        // but the code is not checked. if this fails, then the vault thinks it has allocated to the strategy
        // but in reality it has not
        mUSDC.mint(amount);
        return amount;
    }
```

Looking at the Moonwell contracts, if any operation (mint or redeem) fails for whatever reason, the transaction will not revert but instead return a non-zero error code: (<https://github.com/moonwell-fi/contracts-open-source/blob/e23657c5fbeb12c7393fa49da6f350dc0bd5114e/contracts/core/MErc20.sol#L38-L47>)

```solidity
    /**
     * @notice Sender supplies assets into the market and receives mTokens in exchange
     * @dev Accrues interest whether or not the operation succeeds, unless reverted
     * @param mintAmount The amount of the underlying asset to supply
     * @return uint 0=success, otherwise a failure (see ErrorReporter.sol for details)
     */
    function mint(uint mintAmount) external returns (uint) {
        (uint err,) = mintInternal(mintAmount);
        return err;
    }
```

If allocation fails but the transaction succeeds, the morpho vault still increments assets allocated to this strategy, affecting its perception of caps, both relative and absolute. From VaultV2 (<https://github.com/morpho-org/vault-v2/blob/406546763343b9ffa84c2f63742ae55a490b7c42/src/VaultV2.sol#L571C1-L591C6>)

```solidity
    function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        accrueInterest();

        SafeERC20Lib.safeTransfer(asset, adapter, assets);
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];

   @>         _caps.allocation = (int256(_caps.allocation) + change).toUint256();

            require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());
            require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());
            require(
                _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
                ErrorsLib.RelativeCapExceeded()
            );
        }
        emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
    }
```

This makes the vault think that the underlying assets are generating yield according in line with the strategy risk but in reality the usdc will sit there idle and the protocol will incur an opportunity cost as the share price of MYT will not increase with the intended speed.

In addition, USDC will be sent to the adapter contract but `realAssets()` will in fact report less than the expected amount as there will me less mTokens in the contract:

```
    function realAssets() external view override returns (uint256) {
        // Use stored exchange rate and mToken balance to avoid state changes during static calls
    @>    uint256 mTokenBalance = mUSDC.balanceOf(address(this));
        if (mTokenBalance == 0) return 0;
        uint256 exchangeRate = mUSDC.exchangeRateStored();
        // Exchange rate is scaled by 1e18, so we need to divide by 1e18
        return (mTokenBalance * exchangeRate) / 1e18;
    }
```

Fortunately, idle USDC will still be withdrawable in `deallocate` since the function only checks the before and after balance of USDC underlying

```

    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
        // Pull exact amount of underlying USDC out
        // @audit same silent failure here
        mUSDC.redeemUnderlying(amount);
        uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
        uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
        if (usdcRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(usdc), msg.sender, amount);
        return amount;
    }
```

When deallocating the problem presists since the before and after balance will not satisfy the amount requested, causing unexpected reverts.

## Impact Details

Loss of expected yield since the vault will think that the funds are allocated when in fact they are not. `realAssets` will also report less TVL in the vault even though the funds have been sent but are not reflected as idle. If the Moonwell strategies represent a big proportion of the underlying assets in the vault, eventually this will also cause problems on redemptions of shares since there wont be available liquidity to withdraw assets from the vault (or the vault will think that there is not) , making users unable to realize profits from the yield bearing strategy. In addition, vault state also gets corrupted.

## References

* <https://github.com/morpho-org/vault-v2/blob/main/src/VaultV2.sol>
* <https://github.com/moonwell-fi/contracts-open-source/blob/e23657c5fbeb12c7393fa49da6f350dc0bd5114e/contracts/core/MErc20.sol#L38-L47>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* <https://solodit.cyfrin.io/?b=false\\&f=\\&ff=\\&i=HIGH%2CMEDIUM\\&l=\\&maxf=\\&minf=\\&p=1\\&pc=\\&pn=\\&qs=1\\&r=true\\&rf=alltime\\&rs=1\\&s=moonwell\\&sd=Desc\\&sf=Recency\\&t=\\&u=\\&ur=true>

## Proof of Concept

## Proof of Concept

Add these tests to `MoonwellUSDCStrategy.t.sol`:

1. At the top of the file import:

```
import "forge-std/console.sol";
import {IERC20} from "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
```

```
    function test_POC_moonwell_mint_failure_corrupts_vault_state() public {
        uint256 allocateAmount = 1000e6;

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, allocateAmount);

        // Mock Moonwell mToken to return error code (non-zero = failure)
        vm.mockCall(
            MOONWELL_USDC_MTOKEN,
            abi.encodeWithSignature("mint(uint256)", allocateAmount),
            abi.encode(uint256(5)) // Error code 5 = COMPTROLLER_REJECTION
        );

        bytes memory prevAllocationAmount = abi.encode(0);

        // Strategy allocation "succeeds" despite Moonwell mint failing
        (bytes32[] memory strategyIds, int256 change) = IMYTStrategy(strategy).allocate(
            prevAllocationAmount,
            allocateAmount,
            "",
            address(vault)
        );

        // Vault thinks allocation succeeded - reports full change
        assertEq(change, int256(allocateAmount));
        assertGt(strategyIds.length, 0);

        // But strategy has no real assets - funds trapped in strategy contract
        uint256 realAssets = IMYTStrategy(strategy).realAssets();
        assertEq(realAssets, 0);

        // Strategy contract still holds USDC (allocation failed silently)
        uint256 strategyUSDCBalance = IERC20(testConfig.vaultAsset).balanceOf(strategy);
        assertEq(strategyUSDCBalance, allocateAmount);

        // Clear mock for subsequent operations
        vm.clearMockedCalls();
        vm.stopPrank();

        console.log("Vault believes allocated", uint256(change));
        console.log("Strategy actual real assets", realAssets);
        console.log("USDC still in strategy", strategyUSDCBalance);
    }

    function test_POC_moonwell_redeem_failure_causes_vault_deallocation_to_revert() public {
        uint256 allocateAmount = 1000e6;
        uint256 deallocateAmount = 500e6;

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, allocateAmount);

        // First allocation succeeds normally
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, allocateAmount, "", address(vault));

        // Verify normal allocation worked
        uint256 realAssetsAfterAllocation = IMYTStrategy(strategy).realAssets();
        assertGt(realAssetsAfterAllocation, 0);

        // Mock Moonwell mToken redeemUnderlying to fail silently (return error code)
        vm.mockCall(
            MOONWELL_USDC_MTOKEN,
            abi.encodeWithSignature("redeemUnderlying(uint256)", deallocateAmount),
            abi.encode(uint256(3)) // Error code 3 = TOKEN_INSUFFICIENT_CASH
        );

        bytes memory prevAllocationAmount2 = abi.encode(allocateAmount);

        // Strategy deallocation will REVERT due to insufficient USDC balance
        vm.expectRevert("Strategy balance is less than the amount needed");
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, deallocateAmount, "", address(vault));

        vm.clearMockedCalls();
        vm.stopPrank();

        console.log("Redemption failure causes strategy deallocation to revert");
        console.log("Vault cannot withdraw funds even though allocation tracking shows funds available");
        console.log("Strategy real assets", IMYTStrategy(strategy).realAssets());
        console.log("Amount vault tried to deallocate", deallocateAmount);
    }
```

2. Run the tests

```
forge test --match-test test_POC_moonwell_mint_failure_corrupts_vault_state -vv

forge test --match-test test_POC_moonwell_redeem_failure_causes_vault_deallocation_to_revert -vv
```

You should see that the vault incorrectly thinks that assets are allocated, without yield being generated:

```
Ran 1 test for src/test/strategies/MoonwellUSDCStrategy.t.sol:MoonwellUSDCStrategyTest
[PASS] test_POC_moonwell_redeem_failure_causes_vault_deallocation_to_revert() (gas: 569629)
Logs:
  Redemption failure causes strategy deallocation to revert
  Vault cannot withdraw funds even though allocation tracking shows funds available
  Strategy real assets 999999999
  Amount vault tried to deallocate 500000000

Ran 1 test for src/test/strategies/MoonwellUSDCStrategy.t.sol:MoonwellUSDCStrategyTest
[PASS] test_POC_moonwell_mint_failure_corrupts_vault_state() (gas: 315035)
Logs:
  Vault believes allocated 1000000000
  Strategy actual real assets 0
  USDC still in strategy 1000000000
```
