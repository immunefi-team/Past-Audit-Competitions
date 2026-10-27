# 56911 sc low incorrectly implemented two step admin ownership transfer mechanism prevents new admin to accept role

**Submitted on Oct 21st 2025 at 17:22:19 UTC by @ByteKnight for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56911
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Incorrectly implemented two-step transfer mechanism

## Description

## Brief/Intro

The `acceptAdminOwnership()` function in [AlchemistCurator.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31-L35) has the `onlyAdmin` modifier which prevent new(pending) admin to accept ownership over the smart contract. While the current admin can still complete the transfer by calling `acceptAdminOwnership()` himself, this removes the security benefit of requiring the new admin to explicitly accept ownership, which is designed to prevent accidental transfers to incorrect or non-functional addresses.

## Vulnerability Details

If we take a look at the ownership transfer mechanism in the [AlchemistCurator.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31), we can see that the `acceptAdminOwnership` function has the `onlyAdmin` modifier, which checks that `msg.sender == admin`.

```sol
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
    pendingAdmin = _newAdmin;
}

@> function acceptAdminOwnership() external onlyAdmin {
           admin = pendingAdmin;
           pendingAdmin = address(0);
           emit AdminChanged(admin);
       }
```

However, the intended mechanism for two-step ownership transfer is as follows:

1. Current admin calls the `transferAdminOwnership` and sets a new(pending) admin.
2. The pending admin calls the `acceptAdminOwnership()` function to explicitly accept ownership.

So, the flaw is that the pending admin cannot call the `acceptAdminOwnership()` function because he is not yet an admin and the call will fail. While the current admin can still call the `acceptAdminOwnership()` function to complete the transfer, this defeats the entire purpose of the two-step mechanism, which is to ensure the new admin address is correct and functional before transferring control.

The correct implementation can be found across other project smart contract, for example in the [AlchemistStrategyClassifier.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistStrategyClassifier.sol#L45-L55):

```sol
function transferOwnership(address _newAdmin) external {
    require(msg.sender == admin, "PD");
    pendingAdmin = _newAdmin;
}

function acceptOwnership() external {
    require(msg.sender == pendingAdmin, "PD");
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

## Impact Details

The impacts of this issue are:

* broken two-step ownership transfer. The idea of this mechanism was to prevent accidental transfers to incorrect addresses (e.g., due to typos, copy-paste errors, or sending to a contract address that cannot call functions). By allowing the current admin to complete the transfer unilaterally, this safety mechanism is completely bypassed.
* there is a risk of irrecoverable ownership transfer If the admin mistakenly sets `pendingAdmin` to an incorrect address (wrong address, non-existent EOA, or a contract without the ability to call admin functions), they could still call `acceptAdminOwnership()` and permanently transfer control to an address that cannot manage the contract.
* current implementation will cause unnecessary confusion when pending admin won'y be able to accept ownership and transactions will revert.

While admin transfers can still technically occur (with the current admin calling both functions), the security benefits of the two-step mechanism are completely negated. This creates unnecessary risk during sensitive admin transitions.

## References

* Vulnerable contract: [AlchemistCurator.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31-L35)
* Correct implementation example: [AlchemistStrategyClassifier.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistStrategyClassifier.sol#L45-L55)

## Recommended Fix

Modify the `acceptAdminOwnership()` function to check for `pendingAdmin` instead of using the `onlyAdmin` modifier:

```solidity
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, "PD");
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

## Proof of Concept

## Proof of Concept

To verify the flaw, add the following test to an existing test file AlchemistCurator.t.sol(<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/test/AlchemistCurator.t.sol>). No other changes required in the file. To run the test, run the following script in terminal and paste your fork url.

```
FOUNDRY_PROFILE=default forge test --fork-url <YOUR_FORK_URL> --mt testLegitAcceptAdminOwnershipReverts  -vvvv --evm-version cancun  
```

The test consists of 3 steps:

1. Current admin sets new pending admin by calling the `transferAdminOwnerShip()` function.
2. Pending admin tries to accept the ownership by calling the `acceptAdminOwnership()` function, but transaction reverts.
3. Current admin completes the ownership transfer by calling the `transferAdminOwnerShip()` function himself.

```sol
function testLegitAcceptAdminOwnershipReverts() public {
    // First, current admin initiates ownership transfer to a new admin
    address newAdmin = makeAddr("newAdmin");
    vm.prank(admin);
    mytCuratorProxy.transferAdminOwnerShip(newAdmin);

    // Next, new admin(pending admin) wants to accept the ownership over the Alchemist Curator, but function reverts
    vm.expectRevert(abi.encode("PD"));
    vm.prank(newAdmin);
    mytCuratorProxy.acceptAdminOwnership();

    // Only admin that initiated the ownership transfer can accept pending admin and update the state
    vm.prank(admin);
    mytCuratorProxy.acceptAdminOwnership();
}
```
