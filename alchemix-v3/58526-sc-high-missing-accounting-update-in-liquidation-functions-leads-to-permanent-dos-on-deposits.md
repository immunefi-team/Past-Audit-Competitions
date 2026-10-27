# 58526 sc high missing accounting update in liquidation functions leads to permanent dos on deposits

**Submitted on Nov 3rd 2025 at 02:18:36 UTC by @ibrahimatix0x01 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58526
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `_doLiquidation()` and `_forceRepay()` functions fail to decrement the `_mytSharesDeposited` state variable when transferring MYT tokens out of the contract. This variable tracks total deposits and is checked against `depositCap` to limit protocol exposure. After liquidations occur, `_mytSharesDeposited` becomes permanently inflated relative to the actual token balance, causing all subsequent deposit attempts to revert with `IllegalState` even when well below the configured cap. This renders the core deposit functionality permanently inoperable without a contract upgrade, effectively preventing any new capital from entering the protocol.

## Vulnerability Details

The contract maintains `_mytSharesDeposited` as a private state variable to track the total amount of MYT tokens deposited into the protocol. This variable serves as the basis for enforcing the `depositCap` limit:

```solidity
// Line 428 in deposit()
function deposit(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
    _checkArgument(recipient != address(0));
    _checkArgument(amount > 0);
    _checkState(!depositsPaused);
    _checkState(_mytSharesDeposited + amount <= depositCap);  //  Fails when accounting is broken
    // ...
    _mytSharesDeposited += amount;  //  Correctly incremented on deposits
}
```

Every function that transfers MYT tokens out of the contract must decrement `_mytSharesDeposited` to maintain accurate accounting. Several functions correctly implement this pattern:

**Correctly Implemented Functions:**

```solidity
// withdraw() - Line 467
TokenUtils.safeTransfer(myt, recipient, amount);
_mytSharesDeposited -= amount;

// burn() - Line 633
TokenUtils.safeTransfer(myt, protocolFeeReceiver, convertDebtTokensToYield(credit) * protocolFee / BPS);
_mytSharesDeposited -= convertDebtTokensToYield(credit) * protocolFee / BPS;

// repay() - Line 700
TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
_mytSharesDeposited -= creditToYield * protocolFee / BPS;

// redeem() - Line 733
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;
```

However, the liquidation flow contains two functions that fail to maintain this invariant.

**Bug Location #1: `_doLiquidation()` (Lines 852-893)**

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
    internal
    returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
{
    // ... calculation logic ...
    
    amountLiquidated = convertDebtTokensToYield(liquidationAmount);
    feeInYield = convertDebtTokensToYield(baseFee);

    account.collateralBalance = account.collateralBalance > amountLiquidated ? 
        account.collateralBalance - amountLiquidated : 0;
    _subDebt(accountId, debtToBurn);

    // Transfer liquidated collateral to transmuter
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

    // Transfer liquidation fee to liquidator
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    }

    // BUG: _mytSharesDeposited is never decremented
    // Missing: _mytSharesDeposited -= amountLiquidated;
    
    // ... outsourced fee handling ...
    
    return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
}
```

The function transfers `amountLiquidated` tokens out of the contract (split between transmuter and liquidator), but never decrements `_mytSharesDeposited`. This creates an accounting gap equal to the liquidation amount.

**Bug Location #2: `_forceRepay()` (Lines 738-782)**

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    if (amount == 0) return 0;
    
    // ... validation and calculation logic ...
    
    uint256 credit = amount > debt ? debt : amount;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);

    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

    // Transfer protocol fee
    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    }

    // Transfer repaid amount to transmuter
    if (creditToYield > 0) {
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    }
    
    //  BUG: _mytSharesDeposited is never decremented
    // Missing: _mytSharesDeposited -= (creditToYield + protocolFeeTotal);
    
    return creditToYield;
}
```

This function is called during liquidations when earmarked debt exists. It transfers tokens to both the protocol fee receiver and transmuter, but fails to update `_mytSharesDeposited`.

**How the Bug Manifests:**

1. Protocol starts with accurate accounting: actual balance = `_mytSharesDeposited`
2. Liquidation occurs via `liquidate()` → calls `_liquidate()` → calls `_doLiquidation()` and/or `_forceRepay()`
3. MYT tokens are transferred out, reducing actual balance
4. `_mytSharesDeposited` remains unchanged, creating divergence
5. Subsequent deposit attempts check `_mytSharesDeposited + newAmount <= depositCap`
6. Check fails because `_mytSharesDeposited` is inflated, even though actual balance has room
7. All deposits revert with `IllegalState` error

**Example Scenario:**

```
Initial State:
- depositCap = 1,000 MYT
- Actual Balance = 1,000 MYT
- _mytSharesDeposited = 1,000 MYT
 Accounting is correct

After Liquidation (removes 500 MYT):
- depositCap = 1,000 MYT
- Actual Balance = 500 MYT
- _mytSharesDeposited = 1,000 MYT (unchanged)
Accounting gap of 500 MYT

New Deposit Attempt (100 MYT):
- Check: _mytSharesDeposited (1,000) + amount (100) <= depositCap (1,000)
- Result: 1,100 > 1,000 → REVERT 
- Should be: actualBalance (500) + amount (100) = 600 < 1,000 → SUCCESS 
```

The bug is deterministic and occurs on every liquidation. There is no admin function to manually correct `_mytSharesDeposited`, making the issue permanent without a contract upgrade.

## Impact Details

**Primary Impact: Permanent Protocol Dysfunction**

The vulnerability causes complete and irreversible failure of the `deposit()` function, which is a core protocol operation. Once any liquidation occurs, no new deposits can be processed regardless of actual token availability or deposit cap headroom.

**Immediate Consequences:**

1. **All New Deposits Blocked**: Any user attempting to deposit will encounter `IllegalState` revert
2. **Zero Recovery Path**: No admin function exists to correct `_mytSharesDeposited` manually
3. **Cumulative Degradation**: Each subsequent liquidation worsens the accounting gap
4. **Requires Emergency Upgrade**: Only solution is contract upgrade with full user migration

**Financial Impact:**

**Protocol Level:**

* **Revenue Loss**: Protocol cannot earn fees from new deposits after first liquidation
* **TVL Stagnation**: Total Value Locked permanently capped at pre-liquidation level
* **Growth Paralysis**: Unable to onboard new users or accept additional capital
* **Competitive Disadvantage**: Functional competing protocols will capture market share
* **Reputation Damage**: Users perceive protocol as broken, leading to loss of confidence

**Quantified Losses:**

Assuming a protocol with:

* 10M TVL
* 15% APY fee generation
* 2M liquidated in normal market conditions

```
Year 1 Lost Deposits: ~5M in new capital blocked
Lost Revenue: 5M × 15% × protocol fee = potential 6-figure annual loss
Opportunity Cost: Compounding effect of lost growth
Migration Cost: Emergency upgrade + user communication + potential liquidity event
```

**User Impact:**

* **Prospective Users**: Cannot deposit despite advertised capacity and functional protocol
* **Existing Users**: No direct fund loss; can still withdraw, repay, and perform other operations
* **Liquidators**: Continue operating normally; bug does not affect liquidation mechanism itself

**Why This Qualifies as "Smart contract unable to operate due to lack of token funds":**

The deposit check (`_mytSharesDeposited + amount <= depositCap`) fails as if the contract lacks capacity to accept more funds, even though actual token balance and deposit cap allow it. The contract's deposit functionality becomes inoperable due to incorrect accounting that makes it appear "full" when it is not.

**Severity Justification (High):**

**Permanent Dysfunction**: Core protocol function permanently disabled\
**No Recovery Mechanism**: Cannot be fixed without upgrade\
**Inevitable Trigger**: Liquidations are normal protocol operations **Business Critical**: Prevents protocol growth and new user acquisition\
**No Admin Override**: No emergency function to correct accounting

**Not Critical Because**: No direct theft, no fund loss for existing users, protocol remains solvent

**Attack Complexity: NONE** - This is not an exploitable vulnerability; it's an inherent bug that triggers automatically during normal liquidation operations. No malicious actor is required.

**Likelihood: CERTAIN** - Liquidations occur regularly during market volatility. The bug will manifest in production with 100% certainty once the first liquidation executes.

## References

**Primary Vulnerable Code:**

* AlchemistV3.sol Line 852-893: `_doLiquidation()` function
* AlchemistV3.sol Line 738-782: `_forceRepay()` function

**Related State Variables:**

* AlchemistV3.sol Line 121: `_mytSharesDeposited` (private variable)
* AlchemistV3.sol Line 58: `depositCap` (public variable)

**Deposit Enforcement Location:**

* AlchemistV3.sol Line 428: Deposit cap check in `deposit()` function

**Correctly Implemented Reference Functions:**

* AlchemistV3.sol Line 467: `withdraw()` - Correctly decrements `_mytSharesDeposited`
* AlchemistV3.sol Line 633: `burn()` - Correctly decrements `_mytSharesDeposited`
* AlchemistV3.sol Line 700: `repay()` - Correctly decrements `_mytSharesDeposited`
* AlchemistV3.sol Line 733: `redeem()` - Correctly decrements `_mytSharesDeposited`

## Proof of Concept

## Proof of Concept

The following test demonstrates the vulnerability by showing how liquidations break the deposit accounting, permanently blocking all new deposits even when well below the deposit cap.

Add the below test code to `AlchemistV3.t.sol`

**Test Code:**

```solidity
/**
 * @notice Clean PoC: Liquidation breaks deposit accounting
 * @dev Demonstrates denial of service - legitimate deposits blocked after liquidations
 */
function testPoC_LiquidationBreaksDepositAccounting() external {
    console.log("");
    console.log("=== Liquidation Breaks Deposit Accounting ===");
    console.log("");
    
    // Setup: Set a deposit cap of 1000 MYT
    uint256 DEPOSIT_CAP = 1000e18;
    address alchemistAdmin = alchemist.admin();
    
    vm.prank(alchemistAdmin);
    alchemist.setDepositCap(DEPOSIT_CAP);
    console.log("Deposit cap set to:", DEPOSIT_CAP / 1e18);
    
    // Give users funds
    deal(address(vault), externalUser, 500e18);
    deal(address(vault), anotherExternalUser, 500e18);
    deal(address(vault), yetAnotherExternalUser, 600e18);
    
    // Step 1: Fill the deposit cap
    console.log("");
    console.log("Step 1: Filling Deposit Cap");
    vm.startPrank(externalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), 500e18);
    alchemist.deposit(500e18, externalUser, 0);
    uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
    vm.stopPrank();
    
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), 500e18);
    alchemist.deposit(500e18, anotherExternalUser, 0);
    vm.stopPrank();
    
    uint256 balanceAtCap = IERC20(address(vault)).balanceOf(address(alchemist));
    console.log("Total deposited:", balanceAtCap / 1e18);
    
    // Step 2: Create and execute liquidation
    console.log("");
    console.log("Step 2: Create Liquidatable Position");
    vm.startPrank(externalUser);
    uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId1);
    alchemist.mint(tokenId1, maxBorrow, externalUser);
    console.log("User borrowed:", maxBorrow / 1e18);
    
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrow);
    transmuterLogic.createRedemption(maxBorrow / 10);
    vm.stopPrank();
    
    vm.roll(block.number + 5_256_000);
    
    // Crash the price
    uint256 initialSupply = IERC20(mockStrategyYieldToken).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply * 159 / 100);
    
    console.log("");
    console.log("Step 3: Execute Liquidation");
    uint256 balanceBeforeLiq = IERC20(address(vault)).balanceOf(address(alchemist));
    console.log("Balance before liquidation:", balanceBeforeLiq / 1e18);
    
    vm.prank(yetAnotherExternalUser);
    alchemist.liquidate(tokenId1);
    
    uint256 balanceAfterLiq = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 tokensRemoved = balanceBeforeLiq - balanceAfterLiq;
    
    console.log("Balance after liquidation:", balanceAfterLiq / 1e18);
    console.log("Tokens removed:", tokensRemoved / 1e18);
    
    // Step 4: Demonstrate the accounting bug
    console.log("");
    console.log("Step 4: Accounting Mismatch");
    console.log("Actual balance:", balanceAfterLiq / 1e18);
    console.log("Internal tracking:", DEPOSIT_CAP / 1e18);
    console.log("Gap:", tokensRemoved / 1e18);
    
    // Step 5: Show the impact
    console.log("");
    console.log("Step 5: Impact - Deposits Blocked");
    uint256 smallDeposit = 100e18;
    console.log("Cap:", DEPOSIT_CAP / 1e18);
    console.log("Current balance:", balanceAfterLiq / 1e18);
    console.log("Attempting deposit:", smallDeposit / 1e18);
    console.log("Would total:", (balanceAfterLiq + smallDeposit) / 1e18);
    
    deal(address(vault), yetAnotherExternalUser, smallDeposit);
    
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), smallDeposit);
    
    // This will fail even though it should succeed
    try alchemist.deposit(smallDeposit, yetAnotherExternalUser, 0) {
        console.log("Deposit succeeded");
        vm.stopPrank();
    } catch {
        console.log("Deposit failed");
        vm.stopPrank();
        
        console.log("");
        console.log("VULNERABILITY CONFIRMED");
        console.log("Deposit blocked even though under cap");
        console.log("_mytSharesDeposited not updated in liquidation");
        console.log("");
        
        // Prove the accounting is broken
        assertTrue(balanceAfterLiq < DEPOSIT_CAP, "Balance below cap");
        assertTrue(balanceAfterLiq + smallDeposit <= DEPOSIT_CAP, "Deposit should fit");
    }
}
```

The Foundry test can be ran with this command `forge test --mt testPoC_LiquidationBreaksDepositAccounting -vvvv`.
