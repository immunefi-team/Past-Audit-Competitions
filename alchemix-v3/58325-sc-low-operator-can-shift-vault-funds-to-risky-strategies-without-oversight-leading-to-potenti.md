# 58325 sc low operator can shift vault funds to risky strategies without oversight leading to potential loss of user funds&#x20;

**Submitted on Nov 1st 2025 at 09:34:55 UTC by @Ratt13snak3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58325
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

## **Description:**

The `AlchemistAllocator` contract allows both the `admin`(the DAO) and any assigned `operator` to allocate and deallocate vault funds across investment strategies. However, the current implementation does not enforce any effective limits on the operator’s control over DAO funds.

### **Root Cause:**

The effective enforcement of DAO targets and risk-based limits is bypassed by the line:

```solidity
uint256 daoTarget = type(uint256).max;
```

which renders `adjusted` unrestricted. This design flaw exists in both the `allocate()` and `deallocate()` functions, allowing full operator control. Both functions use the same privilege model and similar logic for computing the `adjusted` allocation amount. However, due to the following issues, an operator can effectively move all DAO funds to any arbitrary strategy, including high-risk or malicious ones:

1. **Lack of per-operator or per-strategy limits:** There is no restriction on how much an operator can allocate or deallocate at a time, as long as it's within the vaults absolute cap.
2. **No DAO oversight or approval mechanism:** The DAO does not need to confirm or approve allocation changes initiated by operators.
3. **Ineffective cap enforcement:** The code compares `absoluteCap` and `relativeCap` to determine the `adjusted` limit, but then overrides this limit with `daoTarget = type(uint256).max`, effectively nullifying any restriction:

   ```solidity
   uint256 daoTarget = type(uint256).max;
   adjusted = adjusted < daoTarget ? adjusted : daoTarget;
   ```
4. **Symmetry with `allocate` vulnerability:** Both `allocate()` and `deallocate()` functions use this same logic, meaning the operator can fully drain one strategy(even one selected by the DAO) and reallocate all vault funds to another of their choosing.

As a result, operators have **DAO-level privileges/Control** over strategy allocation. This could lead to **unauthorized fund movement,** resulting in excessive risk exposure, or loss of assets if the chosen strategy loses funds.

***

## **Impact:**

* **Fund Reallocation Risk:** Operators can bypass DAO-set targets and rebalance the entire vault arbitrarily.
* **Centralization of Control:** An operator can undermine DAO governance by reallocating DAO-managed assets.
* **Loss of Funds:** If a malicious operator or compromised operator address reallocates funds to an unsafe strategy, user funds could be lost.

***

### **Vulnerable Code Snippet:**

```solidity
uint256 daoTarget = type(uint256).max;
uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
if (msg.sender != admin) {
    adjusted = adjusted < daoTarget ? adjusted : daoTarget; 
}
vault.deallocate(adapter, oldAllocation, amount);
```

Because `daoTarget` is set to `type(uint256).max`, the conditional logic never constrains the operator, leaving `adjusted` effectively unlimited.

## Proof of Concept

## Proof of Concept

The following test demonstrates how an operator can deallocate and reallocate all vault funds from one strategy into another.

add the test contract below into the test suite [AlchemistAllocator.t.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/test/AlchemistAllocator.t.sol)

```solidity
contract AlchemistAllocatorOperatorTest is Test {
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
    MockMYTStrategy public riskyStrategy;

    function setUp() public {
        vm.startPrank(admin);
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator);
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        riskyStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "RiskyToken", "RiskyTokenProtocol", IMYTStrategy.RiskClass.HIGH);
        allocator = new MockAlchemistAllocator(address(vault), admin, operator);
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        vault.addAdapter(address(mytStrategy));
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(riskyStrategy)));
        vault.addAdapter(address(riskyStrategy));
        // Set absolute and relative caps for the strategies
        // bytes memory idData = abi.encode("MockTokenProtocol", address(mytStrategy));
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, defaultStrategyAbsoluteCap)));
        vault.increaseAbsoluteCap(idData, defaultStrategyAbsoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, defaultStrategyRelativeCap)));
        vault.increaseRelativeCap(idData, defaultStrategyRelativeCap);
        // bytes memory riskyIdData = abi.encode("RiskyTokenProtocol", address(riskyStrategy));
        bytes memory riskyIdData = riskyStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (riskyIdData, defaultStrategyAbsoluteCap)));
        vault.increaseAbsoluteCap(riskyIdData, defaultStrategyAbsoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (riskyIdData, defaultStrategyRelativeCap)));
        vault.increaseRelativeCap(riskyIdData, defaultStrategyRelativeCap);
        vm.stopPrank();
    }

    function test_OperatorCanDrainAndReallocate() public {
        // Step 1: Deposit user funds into the vault
        _magicDepositToVault(address(vault), user1, 150 ether);

        // Step 2: Admin makes a normal allocation
        vm.startPrank(admin);
        allocator.allocate(address(mytStrategy), 100 ether);
        vm.stopPrank();

        // Step 3: Operator reallocates without any limit
        vm.startPrank(operator);

        // Deallocate entire allocation
        allocator.deallocate(address(mytStrategy), 100 ether);

        // Allocate full vault balance to a risky or attacker-controlled strategy
        allocator.allocate(address(riskyStrategy), 150 ether);

        vm.stopPrank();

        // Step 4: Check that operator was able to reallocate all funds
        bytes32 riskyId = riskyStrategy.adapterId();
        uint256 riskyAllocation = vault.allocation(riskyId);
        assertEq(riskyAllocation, 150 ether, "Operator was able to reallocate full balance");
    }

    function _magicDepositToVault(address _vault, address depositor, uint256 amount) internal {
        deal(address(mockVaultCollateral), address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(address(mockVaultCollateral), address(vault), amount);
        IVaultV2(address(vault)).deposit(amount, address(vault));
        vm.stopPrank();
    }    

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }
}
```

run the test:

```bash
forge test --mt test_OperatorCanDrainAndReallocate
```

**Expected result:** Test passes, showing that the operator successfully reallocated all DAO-managed funds without restriction.
