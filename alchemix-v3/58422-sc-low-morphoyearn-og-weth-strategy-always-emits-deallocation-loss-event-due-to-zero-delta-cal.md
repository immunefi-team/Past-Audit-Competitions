# 58422 sc low morphoyearn og weth strategy always emits deallocation loss event due to zero delta calculation

**Submitted on Nov 2nd 2025 at 08:30:27 UTC by @unineko for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58422
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**

## Description

## 1. Title

MorphoYearn OG WETH strategy always emits deallocation-loss event due to zero delta calculation

**Scope commit:** `a192ab313c81ba3ab621d9ca1ee000110fbdd1e9`

## 2. Description

### Brief/Intro

`MorphoYearnOGWETHStrategy._deallocate` measures the strategy’s WETH balance both before and after a withdraw call **after** the assets have already been returned. As a result, `wethRedeemed` is always zero, causing `StrategyDeallocationLoss` to emit even when the withdraw completes successfully. Worse, if the vault actually returns less than requested, the function reverts after the (incorrect) event and the revert rolls the log back, so no alert is emitted. The signal becomes an anti-signal: guaranteed false positives during healthy operation and no persistent alert when losses really happen.

### Vulnerability Details

* The strategy intends to compare the WETH balance before and after `vault.withdraw(amount, ...)` to detect shortfalls.
* The current implementation calls `TokenUtils.safeBalanceOf(...)` twice **after** withdrawal, so `wethBalanceBefore == wethBalanceAfter`, making `wethRedeemed == 0` regardless of actual redemption.
* The strategy then checks `if (wethRedeemed < amount)` and emits `StrategyDeallocationLoss`, so every deallocation logs a loss event even when the exact amount was returned. This produces constant false positives and erodes operator trust.
* When the vault really returns less than `amount`, the subsequent `require` statements revert, rolling back the event and leaving no trace (false negative). The monitoring hook never records the loss, defeating its purpose.
* The PoC executes both a lossless and a “haircut” deallocation in a mock environment, demonstrating the false-positive / false-negative pair.

### Impact Details

* **Operational noise / monitoring false positives:** Every healthy deallocation triggers a loss alert, overwhelming dashboards and pagers.
* **False negatives for real losses:** When a vault underpays, `_deallocate` reverts after emitting the event, so the log rolls back and the loss goes unreported. Observability is effectively inverted.
* **Automation and debugging friction:** Automated safeguards or incident responders relying on `StrategyDeallocationLoss` cannot distinguish real issues, raising operational risk even without direct fund loss.

### References

* Affected code: `src/strategies/mainnet/MorphoYearnOGWETH.sol::_deallocate`
* PoC: `src/test/L1_MorphoYearnOGWETHLossEvent.t.sol::{testLossEventAlwaysTriggersWithNoActualLoss,testLossEventDoesNotPersistWhenActualLossOccurs}`

### Steps to Reproduce

1. Check out the scoped commit and install dependencies (`forge install`).
2. Run the PoC (Cancun EVM):

   ```bash
   forge test --offline --evm-version cancun \
     --match-contract MorphoYearnOGWETHLossEventPoCTest -vv
   ```
3. Observe that:
   * `StrategyDeallocationLoss` fires even though the mock vault returns the full `AMOUNT` (`testLossEventAlwaysTriggersWithNoActualLoss`).
   * When the vault returns `amount - 1`, `_deallocate` reverts and the event is rolled back (`testLossEventDoesNotPersistWhenActualLossOccurs`).

### Technical Details

```solidity
// src/strategies/mainnet/MorphoYearnOGWETH.sol::_deallocate (current)
vault.withdraw(amount, address(this), address(this));
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore; // always 0
if (wethRedeemed < amount) {
    emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
}
```

* To compute the true delta, `wethBalanceBefore` must be sampled **before** calling `vault.withdraw`.
* Alternatively, the vault’s return value could be trusted as the redeemed amount.
* Without fixing the order of operations, the event is guaranteed to indicate losses on every withdrawal, undermining its purpose.

### Recommended Fix

1. Capture the strategy’s WETH balance prior to calling `vault.withdraw`:

   ```solidity
   uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
   uint256 redeemedShares = vault.withdraw(amount, address(this), address(this));
   uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
   uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
   ```
2. Emit the loss event only when `wethRedeemed < amount`, and decide whether to revert or surface the loss depending on protocol policy.
3. Optionally, reset allowances (to zero) before reuse to align with ERC20 best practices and add regression tests covering both full and partial redemptions.

### Supporting Evidence

* PoC demonstrates the constant loss event emission in a “no-loss” scenario.
* The existing code clearly reads balances after the withdrawal, explaining the zero delta.
* Fixing the measurement order restores meaningful loss detection without affecting strategy flows.

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../MYTStrategy.sol";
import {MorphoYearnOGWETHStrategy} from "../strategies/mainnet/MorphoYearnOGWETH.sol";

contract MockWETH is ERC20 {
    constructor() ERC20("Mock WETH", "mWETH") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function deposit() external payable {
        _mint(msg.sender, msg.value);
    }

    function withdraw(uint256 amount) external {
        _burn(msg.sender, amount);
        payable(msg.sender).transfer(amount);
    }
}

contract MockMorphoVault is ERC20 {
    ERC20 public immutable assetToken;
    uint256 public withdrawHaircut;

    constructor(ERC20 asset_) ERC20("Mock Morpho Vault", "mSHARE") {
        assetToken = asset_;
    }

    function asset() external view returns (address) {
        return address(assetToken);
    }

    function setWithdrawHaircut(uint256 haircut) external {
        withdrawHaircut = haircut;
    }

    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        assetToken.transferFrom(msg.sender, address(this), assets);
        _mint(receiver, assets);
        return assets;
    }

    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares) {
        _burn(owner, assets);
        uint256 transferAmount = assets > withdrawHaircut ? assets - withdrawHaircut : 0;
        assetToken.transfer(receiver, transferAmount);
        return assets;
    }

    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assetsOut) {
        _burn(owner, shares);
        uint256 transferAmount = shares > withdrawHaircut ? shares - withdrawHaircut : 0;
        assetToken.transfer(receiver, transferAmount);
        return shares;
    }

    function convertToAssets(uint256 shares) external pure returns (uint256 assets) {
        return shares;
    }

    function convertToShares(uint256 assets) external pure returns (uint256 shares) {
        return assets;
    }

    function previewWithdraw(uint256 assets) external pure returns (uint256 shares) {
        return assets;
    }
}

contract MorphoYearnOGWETHStrategyHarness is MorphoYearnOGWETHStrategy {
    constructor(
        address _myt,
        IMYTStrategy.StrategyParams memory _params,
        address _vault,
        address _weth,
        address _permit2Address
    ) MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address) {}

    function exposedAllocate(uint256 amount) external returns (uint256) {
        return _allocate(amount);
    }

    function exposedDeallocate(uint256 amount) external returns (uint256) {
        return _deallocate(amount);
    }
}

contract MorphoYearnOGWETHLossEventPoCTest is Test {
    MockWETH internal weth;
    MockMorphoVault internal vault;
    MorphoYearnOGWETHStrategyHarness internal strategy;

    address internal constant FAKE_MORPHO_VAULT = address(0xBEEF);
    uint256 internal constant AMOUNT = 10 ether;

    function setUp() public {
        weth = new MockWETH();
        vault = new MockMorphoVault(weth);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "MorphoOGWETH",
            protocol: "Morpho",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 0
        });

        strategy = new MorphoYearnOGWETHStrategyHarness(
            FAKE_MORPHO_VAULT,
            params,
            address(vault),
            address(weth),
            address(0x1)
        );

        weth.mint(address(strategy), AMOUNT);
    }

    /// @notice `_deallocate` emits StrategyDeallocationLoss even when no loss occurred
    ///         because it samples both balances after the withdraw.
    function testLossEventAlwaysTriggersWithNoActualLoss() public {
        strategy.exposedAllocate(AMOUNT);

        assertEq(weth.balanceOf(address(strategy)), 0, "WETH deposited into vault");
        assertEq(vault.balanceOf(address(strategy)), AMOUNT, "Vault shares minted");

        vm.expectEmit(true, true, true, true, address(strategy));
        emit MYTStrategy.StrategyDeallocationLoss("Strategy deallocation loss.", AMOUNT, 0);

        vm.prank(FAKE_MORPHO_VAULT);
        strategy.exposedDeallocate(AMOUNT);

        assertEq(weth.balanceOf(address(strategy)), AMOUNT, "WETH returned to strategy");
        assertEq(vault.balanceOf(address(strategy)), 0, "Vault shares burned");
    }

    /// @notice When the vault actually returns less than requested, the strategy
    ///         reverts after emitting the loss event, so observers never see it.
    function testLossEventDoesNotPersistWhenActualLossOccurs() public {
        strategy.exposedAllocate(AMOUNT);

        vault.setWithdrawHaircut(1 ether);

        vm.expectRevert("Strategy balance is less than the amount needed");
        vm.prank(FAKE_MORPHO_VAULT);
        strategy.exposedDeallocate(AMOUNT);

        // Event is rolled back because of the revert; no state changes persist.
        assertEq(weth.balanceOf(address(strategy)), 0, "WETH still locked in vault after revert");
        assertEq(vault.balanceOf(address(strategy)), AMOUNT, "Shares remain after revert");
    }
}
```
