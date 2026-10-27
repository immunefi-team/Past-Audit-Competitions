# 56528 sc insight unbounded slippagebps can freeze withdrawals

**Submitted on Oct 17th 2025 at 09:40:39 UTC by @OxPrince for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56528
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`slippageBPS` is accepted from strategy params without validation in `MYTStrategy` and stored both in `params.slippageBPS` and the local copy `slippageBPS`.

* Every concrete strategy subtracts `slippageBPS / 10_000` from withdrawal previews. When the value exceeds 10\_000 (100%), the subtraction underflows and reverts because Solidity 0.8+ guards against underflow.
* The allocator and shared tests rely on `previewAdjustedWithdraw` to size withdrawals. A revert here prevents operators from computing safe withdrawal amounts and blocks standard deallocation workflows.

## Vulnerability Details

Constructor leaves `slippageBPS` unchecked (`src/MYTStrategy.sol:79-99`). No subsequent setter clamps it either.

* Multiple strategies inherit this behaviour; for example:
  * `StargateEthPoolStrategy._previewAdjustedWithdraw` (`src/strategies/optimism/StargateEthPoolStrategy.sol:96-99`)
  * `MoonwellWETHStrategy._previewAdjustedWithdraw` (`src/strategies/optimism/MoonwellWETHStrategy.sol:81-89`)
  * `FluidARBUSDCStrategy._previewAdjustedWithdraw` (`src/strategies/arbitrum/FluidARBUSDCStrategy.sol:51-55`)
* Shared test helpers and the allocator code path call `previewAdjustedWithdraw` before deallocations (`src/test/libraries/BaseStrategyTest.sol:140-151`, `src/AlchemistAllocator.sol:46-65`), so any revert there halts the withdrawal flow.

## Impact Details

* Misconfigured deployments with `slippageBPS > 10_000` make every `previewAdjustedWithdraw` call revert, preventing vault operators from calculating deallocation amounts through the intended interface.
* Because the allocator is expected to follow the documented flow, withdrawals become operationally blocked until the contract is upgraded or replaced, effectively freezing funds managed by that strategy.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {stdError} from "forge-std/StdError.sol";
import {MYTStrategy} from "../MYTStrategy.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MockERC20} from "solmate/test/utils/mocks/MockERC20.sol";

contract MYTStrategyBaseTest is Test {
    MockERC20 internal asset;
    MockERC20 internal receipt;
    MockVault internal vault;
    MYTStrategy internal strategy;

    function setUp() public {
        asset = new MockERC20("Asset", "ASSET", 18);
        receipt = new MockERC20("Receipt", "RECEIPT", 18);
        vault = new MockVault(asset);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "Base Strategy",
            protocol: "Test Protocol",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 0,
            globalCap: 0,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 0
        });

        strategy = new MYTStrategy(address(vault), params, address(0x1234), address(receipt));
    }


    function test_previewAdjustedWithdrawRevertsWhenSlippageExceedsOneHundredPercent() public {
        IMYTStrategy.StrategyParams memory badParams = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "Bad Slippage Strategy",
            protocol: "Test Protocol",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 0,
            globalCap: 0,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 20_000
        });

        SlippagePreviewStrategy badStrategy = new SlippagePreviewStrategy(address(vault), badParams, address(0x1234), address(receipt));

        vm.expectRevert(stdError.arithmeticError);
        badStrategy.previewAdjustedWithdraw(1 ether);
    }
}



contract SlippagePreviewStrategy is MYTStrategy {
    constructor(address _myt, StrategyParams memory _params, address _permit2Address, address _receiptToken)
        MYTStrategy(_myt, _params, _permit2Address, _receiptToken)
    {}

    function _previewAdjustedWithdraw(uint256 amount) internal view override returns (uint256) {
        return amount - (amount * slippageBPS / 10_000);
    }
}

contract MockVault {
    MockERC20 public immutable asset;
    mapping(address => uint256) private allocations;

    constructor(MockERC20 _asset) {
        asset = _asset;
    }

    function allocate(address strategy, uint256 amount) external {
        bytes memory data = abi.encode(allocations[strategy]);
        asset.transfer(strategy, amount);
        (, int256 change) = IMYTStrategy(strategy).allocate(data, amount, this.allocate.selector, msg.sender);
        allocations[strategy] = _applyChange(allocations[strategy], change);
    }

    function deallocate(address strategy, uint256 amount) external {
        bytes memory data = abi.encode(allocations[strategy]);
        (, int256 change) = IMYTStrategy(strategy).deallocate(data, amount, this.deallocate.selector, msg.sender);
        allocations[strategy] = _applyChange(allocations[strategy], change);
        asset.transferFrom(strategy, address(this), amount);
    }

    function currentAllocation(address strategy) external view returns (uint256) {
        return allocations[strategy];
    }

    function _applyChange(uint256 current, int256 change) internal pure returns (uint256) {
        if (change > 0) {
            return current + uint256(change);
        } else if (change < 0) {
            return current - uint256(-change);
        }
        return current;
    }
}

```

forge test --match-test previewAdjustedWithdrawRevertsWhenSlippageExceedsOneHundredPercent -vvvv

│ │ └─ ← \[Return] true │ └─ ← \[Return] 7080 bytes of code ├─ \[0] VM::expectRevert(custom error 0xf28dceb3: $NH{q) │ └─ ← \[Return] ├─ \[1637] SlippagePreviewStrategy::previewAdjustedWithdraw(1000000000000000000 \[1e18]) \[staticcall] │ └─ ← \[Revert] panic: arithmetic underflow or overflow (0x11)
