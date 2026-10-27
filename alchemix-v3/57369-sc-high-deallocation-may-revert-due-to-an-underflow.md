# 57369 sc high deallocation may revert due to an underflow

**Submitted on Oct 25th 2025 at 15:16:47 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57369
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The function `deallocate` may revert due to an underflow when the `amountDeallocated` is greater than `oldAllocation`.

## Vulnerability Details

The function `deallocate` is used to remove funds from strategies by calling internal `deallocate` function with the specified asset amount. The function then attemps to calculate the new allocation by subtracting the amount deallocated from `oldAllocation`.

The vulnerability arises because `amountDeallocated`, which is returned by `_deallocate`, can potentially be greater than `oldAllocation`. This may happen when `_deallocate` withdraws more funds than expected. And when `amountDeallocated` exceeds `oldAllocation`, the tx would revert due to underflow.

Code snippet:

```solidity
    function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0));
        }
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountDeallocated = _deallocate(assets);
        uint256 newAllocation = oldAllocation - amountDeallocated;

        emit Deallocate(amountDeallocated, address(this));
        return (ids(), int256(newAllocation) - int256(oldAllocation));
    }
```

## Impact Details

The impact is temporary freezing of funds, because the admin would not be able to deallocate funds from strategies. This is a potential Denial of Service.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L130>

## Proof of Concept

## Proof of Concept

Create a new file in path `src/test/PoC.t.sol`:

Run with: `forge test --match-test test_underflow -vvvv`

PoC:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";

interface IVaultV2 {
    function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        returns (bytes32[] memory strategyIds, int256 change);

    function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        returns (bytes32[] memory strategyIds, int256 change);
}

/// @title PoC Strategy that underflows on deallocate
contract PoCStrategy {

    IVaultV2 public immutable vault;

    constructor(address _vault) {
        vault = IVaultV2(_vault);
    }

    /// @notice minimal deallocate logic that intentionally underflows
    function deallocate(bytes memory data, uint256 assets, bytes4, address) external returns (bytes32[] memory, int256) {
        // decode old allocation
        uint256 oldAllocation = abi.decode(data, (uint256));

        // _deallocate returns more than oldAllocation to force underflow
        uint256 amountDeallocated = _deallocate(assets);

        // intentional underflow
        uint256 newAllocation = oldAllocation - amountDeallocated;

        bytes32[] memory ids_ = new bytes32[](1);
        ids_[0] = bytes32("PoC");
    
        return (ids_, int256(newAllocation) - int256(oldAllocation));
    }

    function _deallocate(uint256 assets) internal pure returns (uint256) {
        return assets * 2; // deliberately larger than oldAllocation
    }
}

/// @title Minimal Vault Mock
contract VaultMock is IVaultV2 {
    PoCStrategy public strategy;

    constructor() {}

    function setStrategy(address _strategy) external {
        strategy = PoCStrategy(_strategy);
    }

    function allocate(bytes memory, uint256, bytes4, address) external pure override returns (bytes32[] memory, int256) {
        bytes32[] memory ids_ = new bytes32[](1);
        ids_[0] = bytes32("ALLOC");
        return (ids_, 0);
    }

    function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender) external override returns (bytes32[] memory strategyIds, int256 change) {
        return strategy.deallocate(data, assets, selector, sender);
    }
}

/// @title PoC Test
contract PoCUnderflowTest is Test {

    VaultMock public vault;
    PoCStrategy public strategy;

    function setUp() public {
        vault = new VaultMock();
        strategy = new PoCStrategy(address(vault));
        vault.setStrategy(address(strategy));
    }

    function test_underflow() public {
        uint256 oldAllocation = 50;
        bytes memory data = abi.encode(oldAllocation);

        // assets = 100, _deallocate will return 200, causing underflow
        uint256 assets = 100;

        // Expect Panic(0x11) due to underflow
        vm.expectRevert();
        vault.deallocate(data, assets, bytes4(0), address(this));
    }
}
```

Result:

```solidity
[PASS] test_underflow() (gas: 20449)
Traces:
  [20449] PoCUnderflowTest::test_underflow()
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [8137] VaultMock::deallocate(0x0000000000000000000000000000000000000000000000000000000000000032, 100, 0x00000000, PoCUnderflowTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ [1514] PoCStrategy::deallocate(0x0000000000000000000000000000000000000000000000000000000000000032, 100, 0x00000000, PoCUnderflowTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped;
```
