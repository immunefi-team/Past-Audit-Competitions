# 56956 sc high lack of slippage control in tokemak strategies can make myt suffer losses on allocation

**Submitted on Oct 22nd 2025 at 07:11:38 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56956
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Lack of slippage control leading to loss of funds

## Description

## Brief/Intro

`TokeAutoEthStrategy` , upon an `_allocate` call, deposits assets to the `AutopilotRouter` to obtain shares and then stakes the shares to start generating rewards. Upon depositing however, the `minSharesOut` is hardcoded to 0 in the `depositMax` call which could lead to loss of funds due to on-chain slippage. The same issue applies to `TokeAutoUsdStrategy` as well.

## Vulnerability Details

`_allocate` in `TokeAutoEthStrategy` calls `depositMax` with a `minSharesOut` always set to 0.

```solidity
    // @dev Implementation can alternatively make use of a multicall
    // Deposit weth into the autoEth vault, stake the shares in the rewarder
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(router), amount);
        // @audit min shares out are 0, lack of slippage control
        // https://docs.auto.finance/developer-docs/integrating/4626-compliance#slippage
@>  uint256 shares = router.depositMax(autoEth, address(this), 0);
        TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
        // @audit shares are never checked
        rewarder.stake(address(this), shares);
        return amount;
    }
```

As stated by the Tokemak documentation (<https://docs.auto.finance/developer-docs/integrating/4626-compliance#slippage>),

```
Depending on the conditions of the Autopool, the overall market, and the timing of the debt reporting process slippage may be encountered on both entering and exiting the Autopool. It is very important to always check the shares received on entering, and the assets received on exiting, are greater than an expected amount. 
```

The code however never checks the shares received against an expected amount leaving allocations vulnerable to on-chain slippage which can occur naturally due to liquidity issues or mev epecially as the size of the allocation scales.

## Impact Details

Shares received might be fewer than expected which can in turn redeem fewer assets than the amount allocated which causes loss of funds, potentially substancial.

## References

<https://docs.auto.finance/developer-docs/integrating/4626-compliance#slippage>

## Proof of Concept

## Proof of Concept

We will test `TokeAutoEthStrategy` here but the same issue can be replicated for the USD strategy as well.

1. In `TokeAutoEthStrategy.t.sol` add these lines before the contract declaration to import the required dependencies

```solidity
import "forge-std/console.sol";
import {IERC4626} from "../../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
import {IMainRewarder, IAutopilotRouter} from "../../strategies/interfaces/ITokemac.sol";
```

2. Then at the bottom of the file add this poc and helper function that simulates on-chain slippage.

```solidity
    /// @notice PoC: Demonstrates slippage vulnerability in TokeAutoEth allocation
    function test_POC_allocation_slippage_no_protection() public {
        uint256 amountToAllocate = 10e18; // 10 WETH

        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);

        uint256 expectedShares = IERC4626(TOKE_AUTO_ETH_VAULT).previewDeposit(amountToAllocate);
        console.log("Current rate without slippage:", expectedShares);

        // Allocate through strategy (calls router.depositMax with minSharesOut = 0)
        console.log("Allocating without slippage protection...");
        bytes memory prevAllocationAmount = abi.encode(0);
        mockSlippage(expectedShares, 1000); // 10% slippage
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        uint256 actualShares = MockTokeAutoEthStrategy(strategy).rewarder().balanceOf(strategy);
        console.log("Actual shares received:  ", actualShares);
        
        assertLt(actualShares, expectedShares, "Actual shares should be less than expected shares");

        vm.stopPrank();
    }

    function mockSlippage(uint256 expectedShares, uint256 slippageBps) public {
        // mock call the depositMax function with less than expected shares
        console.log("Mocking slippage of", slippageBps, "bps due to on-chain conditions and order size");
        uint256 sharesWithSlippage = expectedShares - (expectedShares * slippageBps / 10_000);
        vm.mockCall(
            AUTOPILOT_ROUTER,
            abi.encodeWithSelector(IAutopilotRouter.depositMax.selector, TOKE_AUTO_ETH_VAULT, address(strategy), 0),
            abi.encode(sharesWithSlippage)
        );
        // deal the expected shares with slippage so that there is balance to stake
        deal(TOKE_AUTO_ETH_VAULT, address(strategy), sharesWithSlippage);
    }
```

3. Run the test:

```
forge test --match-test test_POC_allocation_slippage_no_protection -vv
```

The test should pass successfully. As we can see, we mocked a slippage of 10% and received less shares than expected but the code did not perform any checks and the call succeeded. You should see this output in your console:

```
[PASS] test_POC_allocation_slippage_no_protection() (gas: 656148)
Logs:
  Current rate without slippage: 9729328543427726812
  Allocating without slippage protection...
  Mocking slippage of  1000 bps due to on-chain conditions and order size
  Shares with slippage:  8756395689084954131
  Actual shares received:   8756395689084954131
```

Slippage can occur for many reasons like tokemak outlines in their documentation since the underlying vaults perform complex strategies which could be subject to liquidity constraints.
