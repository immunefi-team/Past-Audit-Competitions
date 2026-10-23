# 56909 sc low incorrect balance snapshot in strategy deallocation causes false loss events and masks real shortfalls

**Submitted on Oct 21st 2025 at 17:15:26 UTC by @ihtishamsudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56909
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The deallocation logic in MorphoYearnOGWETHStrategy measures “before” and “after” balances after the withdrawal operation, causing the strategy to compute zero redeemed assets on every deallocation.

## Vulnerability Details

Both wethBalanceBefore and wethBalanceAfter are sampled after vault.withdraw. Therefore, wethRedeemed is computed as 0 every time, regardless of what the vault actually returned.

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this)); //@audit incorrect order
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

## Impact Details

StrategyDeallocationLoss is emitted on every deallocation with misleading data (looks like total loss: actualAmountSent=0) and actual redeemed funds are never measured correctly.

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49>

## Proof of Concept

## Proof of Concept

Add following in MorphoYearnOGWETHStrategy.t.sol and execute test with `forge test --mt test_bug_emits_false_loss_event -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../libraries/BaseStrategyTest.sol";
import "forge-std/Test.sol";
import {MorphoYearnOGWETHStrategy} from "../../strategies/mainnet/MorphoYearnOGWETH.sol";

// Minimal ERC20 mock for testing
import {ERC20} from "../../../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract ERC20Mock is ERC20 {
    constructor(string memory name_, string memory symbol_) ERC20(name_, symbol_) {}
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

// Minimal ERC4626-like mock implementing only what the strategy uses
contract ERC4626Mock {
    ERC20 public immutable assetToken;
    mapping(address => uint256) internal _shares;

    constructor(address asset_) {
        assetToken = ERC20(asset_);
    }

    function asset() external view returns (address) {
        return address(assetToken);
    }

    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        // pull assets from msg.sender (strategy must have approved this contract)
        require(assetToken.transferFrom(msg.sender, address(this), assets), "transferFrom failed");
        shares = assets;
        _shares[receiver] += shares;
    }

    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares) {
        // burn shares from owner and send assets to receiver
        shares = assets;
        require(_shares[owner] >= shares, "insufficient shares");
        _shares[owner] -= shares;
        require(assetToken.transfer(receiver, assets), "transfer failed");
    }

    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assetsOut) {
        assetsOut = shares;
        require(_shares[owner] >= shares, "insufficient shares");
        _shares[owner] -= shares;
        require(assetToken.transfer(receiver, assetsOut), "transfer failed");
    }

    function convertToAssets(uint256 shares) external pure returns (uint256 assets) {
        return shares;
    }

    function convertToShares(uint256 assets) external pure returns (uint256 shares) {
        return assets;
    }

    function balanceOf(address account) external view returns (uint256) {
        return _shares[account];
    }

    function previewWithdraw(uint256 assets) external pure returns (uint256 shares) {
        return assets;
    }
}

contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address)
    {}
}

contract MorphoYearnOGWETHStrategyTest is BaseStrategyTest {
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "MorphoYearnOGETH",
            protocol: "MorphoYearnOGETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 23_298_447;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockMorphoYearnOGWETHStrategy(vault, params, MORPHO_YEARN_OG_VAULT, WETH, MAINNET_PERMIT2));
    }

    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

    /*     

    function test_allocated_position_generated_yield() public {
            vm.startPrank(address(vault));
            uint256 amount = 100 ether;
            deal(WETH, address(mytStrategy), amount);
            bytes memory prevAllocationAmount = abi.encode(0);
            mytStrategy.allocate(prevAllocationAmount, amount, "", address(vault));
            uint256 initialRealAssets = mytStrategy.realAssets();
            emit MorphoYearnOGWETHStrategyTestLog("initialRealAssets", initialRealAssets);
            assertApproxEqAbs(initialRealAssets, amount, 1e18);
            vm.warp(block.timestamp + 180 days);
            uint256 realAssets = mytStrategy.realAssets();
            emit MorphoYearnOGWETHStrategyTestLog("realAssets", realAssets);
            assertGt(realAssets, initialRealAssets);
            vm.stopPrank();
        }
    */
}

contract MorphoYearnOGWETHStrategyBugDemoTest is Test {
    // mirror the event signature from MYTStrategy to assert on it
    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

    function test_bug_emits_false_loss_event() public {
        // Arrange: set up mock WETH, mock 4626 vault, and strategy
        address myt = address(0xBEEF);
        ERC20Mock weth = new ERC20Mock("WETH", "WETH");
        ERC4626Mock vault = new ERC4626Mock(address(weth));

        // Build minimal params
        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "MorphoYearnOGETH",
            protocol: "MorphoYearnOGETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });

        MorphoYearnOGWETHStrategy strat = new MockMorphoYearnOGWETHStrategy(
            myt, params, address(vault), address(weth), address(0x01)
        );

        // Fund the strategy so it can deposit
        uint256 amount = 10 ether;
        weth.mint(address(strat), amount);

        // Act: allocate and then deallocate as the MYT (vault)
        // Expectation: due to the bug, deallocate emits a false loss with actualAmountSent == 0
        vm.startPrank(myt);
        IMYTStrategy(address(strat)).allocate(abi.encode(0), amount, "", myt);

        vm.expectEmit(false, false, false, true);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, 0);
        IMYTStrategy(address(strat)).deallocate(abi.encode(amount), amount, "", myt);
        vm.stopPrank();
    }
}
```

* Test logs

```bash
forge test --mt test_bug_emits_false_loss_event -vvv
[⠒] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/strategies/MorphoYearnOGWETHStrategy.t.sol:MorphoYearnOGWETHStrategyBugDemoTest
[PASS] test_bug_emits_false_loss_event() (gas: 3303480)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.85ms (2.86ms CPU time)

Ran 1 test suite in 16.82ms (4.85ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
