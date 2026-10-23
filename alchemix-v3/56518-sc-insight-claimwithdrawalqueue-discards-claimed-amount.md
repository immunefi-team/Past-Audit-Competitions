# 56518 sc insight claimwithdrawalqueue discards claimed amount

**Submitted on Oct 17th 2025 at 07:50:56 UTC by @OxPrince for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56518
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

`MYTStrategy.claimWithdrawalQueue` declares a return value but drops the amount produced by the strategy-specific `_claimWithdrawalQueue`. Extenders such as `SfrxETHStrategy` correctly compute and return the assets they redeemed, yet callers of the public entrypoint always receive `0`.

## Vulnerability Details

`claimWithdrawalQueue` at `src/MYTStrategy.sol:143-148` does not capture the value returned by `_claimWithdrawalQueue`, so the Solidity default return value `0` is forwarded. The interface `IMYTStrategy.claimWithdrawalQueue` explicitly promises a `uint256` return, and concrete strategies (e.g., `SfrxETHStrategy._claimWithdrawalQueue` at `src/strategies/SfrxETH.sol:72-82`) produce a meaningful amount that is never surfaced to the caller.

## Impact Details

vault accounting that relies on the returned amount (per the `IMYTStrategy` interface) will conclude that nothing was claimed, even while assets were actually withdrawn. That leads to stalled exit flows, double-withdraw attempts, or failure to re-integrate newly received assets into vault accounting. In the worst case, downstream code may revert because it believes there are no funds to settle, effectively freezing allocators until manual intervention.

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

    function test_baseHooksLeaveVaultUnableToRecoverAssets() public {
        uint256 amount = 10 ether;

        asset.mint(address(vault), amount);

        vault.allocate(address(strategy), amount);

        assertEq(asset.balanceOf(address(strategy)), amount, "strategy did not retain the transfer");
        assertEq(vault.currentAllocation(address(strategy)), 0, "allocation should remain zero with empty hooks");

        vm.expectRevert(stdError.arithmeticError);
        vault.deallocate(address(strategy), amount);
    }
}

contract ClaimValueStrategy is MYTStrategy {
    uint256 public claimValue;
    uint256 public lastInternalClaim;

    constructor(address _myt, StrategyParams memory _params, address _permit2Address, address _receiptToken)
        MYTStrategy(_myt, _params, _permit2Address, _receiptToken)
    {}

    function setClaimValue(uint256 newValue) external {
        claimValue = newValue;
    }

    function _claimWithdrawalQueue(uint256) internal override returns (uint256) {
        lastInternalClaim = claimValue;
        return claimValue;
    }
}

contract ClaimWithdrawalQueueReturnTest is Test {
    ClaimValueStrategy internal strategy;

    function setUp() public {
        MockERC20 asset = new MockERC20("Asset", "ASSET", 18);
        MockVault vault = new MockVault(asset);
        MockERC20 receipt = new MockERC20("Receipt", "RECEIPT", 18);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "Claim Strategy",
            protocol: "Test Protocol",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 0,
            globalCap: 0,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 0
        });

        strategy = new ClaimValueStrategy(address(vault), params, address(0x1234), address(receipt));
        strategy.setWhitelistedAllocator(address(this), true);
    }

    function test_claimWithdrawalQueue_returnsZero() public {
        strategy.setClaimValue(5 ether);

        uint256 returnedAmount = strategy.claimWithdrawalQueue(1);

        assertEq(strategy.lastInternalClaim(), 5 ether, "internal hook should record claimed value");
        assertEq(returnedAmount, 0, "external call incorrectly returns zero");
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
