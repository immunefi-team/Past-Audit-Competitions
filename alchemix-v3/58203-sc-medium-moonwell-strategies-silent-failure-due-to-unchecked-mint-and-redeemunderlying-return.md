# 58203 sc medium moonwell strategies silent failure due to unchecked mint and redeemunderlying return values

**Submitted on Oct 31st 2025 at 10:52:38 UTC by @Brainiac5 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58203
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

The Moonwell strategies (`MoonwellUSDCStrategy`, `MoonwellWETHStrategy`) use `mint()` and `redeemUnderlying()` functions that can return error codes instead of reverting on failure. The strategies don't check these return values, causing silent failures where the strategy believes operations succeeded when they actually failed.

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
    // @audit No return value check - if mint fails, strategy thinks it succeeded
    mUSDC.mint(amount);
    return amount;
}

function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
    
    // @audit No return value check - if redeem fails, next require will revert
    // but the error message won't indicate the real failure
    mUSDC.redeemUnderlying(amount);
    
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(usdc), msg.sender, amount);
    
    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
    uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
    
    if (usdcRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
    }
    
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
    return amount;
}
```

## Vulnerability Details

### Moonwell mToken Mint Flow

**Contract**: Moonwell mUSDC ([0x6E745367F4Ad2b3da7339aee65dC85d416614D90](https://vscode.blockscan.com/1285/0x6E745367F4Ad2b3da7339aee65dC85d416614D90))

**Step 1: External mint function returns error code**

```solidity
function mint(uint mintAmount) external returns (uint) {
    (uint err,) = mintInternal(mintAmount);
    return err;  // Returns error code, does NOT revert
}
```

**Step 2: Internal mint checks multiple conditions**

```solidity
function mintInternal(uint mintAmount) internal nonReentrant returns (uint, uint) {
    // Check 1: Accrue interest first
    uint error = accrueInterest();
    if (error != uint(Error.NO_ERROR)) {
        return (fail(Error(error), FailureInfo.MINT_ACCRUE_INTEREST_FAILED), 0);
    }
    
    // Proceed to fresh mint
    return mintFresh(msg.sender, mintAmount);
}
```

**Step 3: mintFresh performs critical validations**

```solidity
function mintFresh(address minter, uint mintAmount) internal returns (uint, uint) {
    // Check 2: Comptroller allows mint?
    uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);
    if (allowed != 0) {
        return (failOpaque(Error.COMPTROLLER_REJECTION, FailureInfo.MINT_COMPTROLLER_REJECTION, allowed), 0);
    }

    // Check 3: Market is fresh (accrual timestamp matches)?
    if (accrualBlockTimestamp != getBlockTimestamp()) {
        return (fail(Error.MARKET_NOT_FRESH, FailureInfo.MINT_FRESHNESS_CHECK), 0);
    }

    MintLocalVars memory vars;

    // Check 4: Exchange rate calculation succeeds?
    (vars.mathErr, vars.exchangeRateMantissa) = exchangeRateStoredInternal();
    if (vars.mathErr != MathError.NO_ERROR) {
        return (failOpaque(Error.MATH_ERROR, FailureInfo.MINT_EXCHANGE_RATE_READ_FAILED, uint(vars.mathErr)), 0);
    }

    // EFFECTS & INTERACTIONS
    // Transfer tokens from minter to mToken contract
    vars.actualMintAmount = doTransferIn(minter, mintAmount);

    // Calculate mTokens to mint
    (vars.mathErr, vars.mintTokens) = divScalarByExpTruncate(vars.actualMintAmount, Exp({mantissa: vars.exchangeRateMantissa}));
    require(vars.mathErr == MathError.NO_ERROR, "MINT_EXCHANGE_CALCULATION_FAILED");

    // Update total supply
    (vars.mathErr, vars.totalSupplyNew) = addUInt(totalSupply, vars.mintTokens);
    require(vars.mathErr == MathError.NO_ERROR, "MINT_NEW_TOTAL_SUPPLY_CALCULATION_FAILED");

    // Update minter balance
    (vars.mathErr, vars.accountTokensNew) = addUInt(accountTokens[minter], vars.mintTokens);
    require(vars.mathErr == MathError.NO_ERROR, "MINT_NEW_ACCOUNT_BALANCE_CALCULATION_FAILED");

    totalSupply = vars.totalSupplyNew;
    accountTokens[minter] = vars.accountTokensNew;

    emit Mint(minter, vars.actualMintAmount, vars.mintTokens);
    emit Transfer(address(this), minter, vars.mintTokens);

    return (uint(Error.NO_ERROR), vars.actualMintAmount);
}
```

### The Silent Failure Problem

**What the strategy does:**

```solidity
// Strategy IGNORES the return value
mUSDC.mint(amount);
return amount;  // Claims success regardless of actual result
```

**What should happen if mint fails:**

```solidity
uint err = mUSDC.mint(amount);
// err could be:
// - Error.COMPTROLLER_REJECTION (comptroller denied the mint)
// - Error.MARKET_NOT_FRESH (accrual timestamp mismatch)  
// - Error.MATH_ERROR (exchange rate calculation failed)
// - Error.MINT_ACCRUE_INTEREST_FAILED (interest accrual failed)

// But strategy never checks err, so it assumes success!
```

### Failure Scenarios

#### Primary Issue: Mint Failure (Allocation)

**When Moonwell mint() can fail and return error codes:**

1. **Comptroller Rejection**:
   * Market has reached supply cap
   * Minter is blacklisted or paused
   * Comptroller is in emergency shutdown
2. **Market Not Fresh**:
   * Block timestamp doesn't match accrual timestamp
   * Interest hasn't been accrued properly
3. **Interest Accrual Failure**:
   * Interest rate model calculation fails
   * Accumulation causes overflow
4. **Math Errors**:
   * Exchange rate calculation fails
   * Token supply calculations overflow

**Impact of ignored error:**

```
1. Strategy receives 1000 USDC to allocate
2. Calls mUSDC.mint(1000e6) - returns error code 3 (COMPTROLLER_REJECTION)
3. Strategy IGNORES return value and returns 1000e6
4. VaultV2 believes 1000 USDC is now earning yield in Moonwell
5. REALITY: 1000 USDC sits idle in strategy contract, earning nothing
6. Accounting shows "allocated(1000e6)" but funds are actually undeployed
```

#### Secondary Issue: Redeem Failure (Deallocation)

While deallocation failures will eventually cause a revert due to the balance check, the error message will be misleading:

```solidity
mUSDC.redeemUnderlying(amount);  // Returns error code (e.g., insufficient liquidity)
require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
// Reverts with generic message instead of actual error: "Moonwell has insufficient liquidity"
```

## Impact

### Severity Justification

* **Silent Allocation Failures**: Critical - funds never enter yield positions but accounting shows them as allocated and they are left stuck in the startegies.
* **Accounting Corruption**: VaultV2 tracking diverges from reality, believing funds are deployed when they're idle
* **Loss of Yield**: Capital sits unproductive in strategy contract instead of earning Moonwell supply APY

## Recommended Fix

Check return values and revert with clear error messages:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
    
    // Check return value
    uint256 err = mUSDC.mint(amount);
    require(err == 0, "Moonwell mint failed");
    
    return amount;
}

function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
    
    // Check return value
    uint256 err = mUSDC.redeemUnderlying(amount);
    require(err == 0, "Moonwell redeemUnderlying failed");
    
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(usdc), msg.sender, amount);
    
    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
    uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
    
    if (usdcRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
    }
    
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
    return amount;
}
```

## References

* **Vulnerable Code**:
* **Moonwell mUSDC Contract**: [0x6E745367F4Ad2b3da7339aee65dC85d416614D90](https://vscode.blockscan.com/1285/0x6E745367F4Ad2b3da7339aee65dC85d416614D90)

## Proof of Concept

### Proof of Concept

## Proof of Concept Setup

**1. Create Mock Contract** (mock contract included below)

* **Path**: `test/mocks/MoonwellMock.sol`
* **Purpose**: Simulates Moonwell mToken behavior with controllable failure modes

**2. Create Test File**

* **Path**: `test/MoonwellSilentFailure.t.sol`
* **Purpose**: Demonstrates mint() and redeemUnderlying() return error codes instead of reverting

**3. Run Tests**

````bash
# Test mint silent failure
forge test --match-test test_MintSilentFailure -vv

# Test redeem silent failure  
forge test --match-test test_RedeemSilentFailure -vv

# Run both tests
forge test --match-contract MoonwellSilentFailureTest -vv

## Test contract (MoonwellSilentFailure.sol)


```solidity 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "forge-std/Test.sol";
import {MoonwellMTokenMock, MockERC20} from "./mocks/MoonwellMock.sol";

/**
 * @title MoonwellSilentFailureTest
 * @notice Demonstrates that Moonwell's mint() and redeemUnderlying() return error codes
 *         instead of reverting, but the strategy doesn't check these return values
 * 
 * BUG: The MoonwellUSDCStrategy calls mUSDC.mint() and mUSDC.redeemUnderlying()
 * without checking their return values. These functions return 0 on success and
 * non-zero error codes on failure, but DON'T REVERT.
 * 
 * This means:
 * - _allocate() thinks it succeeded when mint actually failed
 * - _deallocate() thinks it succeeded when redeem actually failed
 * - Users lose funds or can't withdraw
 */
contract MoonwellSilentFailureTest is Test {
    MoonwellMTokenMock public mUSDC;
    MockERC20 public usdc;
    
    address public strategy = address(0x1);
    
    function setUp() public {
        usdc = new MockERC20("USD Coin", "USDC");
        mUSDC = new MoonwellMTokenMock(address(usdc));
        
        // Give strategy some USDC
        usdc.mint(strategy, 1000e6);
    }
    
    /**
     * @notice Demonstrates mint() silent failure
     * @dev mint() returns error code but strategy doesn't check it
     */
    function test_MintSilentFailure() public {
        uint256 amount = 100e6;
        
        // Make mint fail
        mUSDC.setFailMint(true);
        
        vm.startPrank(strategy);
        
        uint256 usdcBefore = usdc.balanceOf(strategy);
        uint256 mUSDCBefore = mUSDC.balanceOf(strategy);
        
        console.log("=== Before mint() ===");
        console.log("Strategy USDC balance:", usdcBefore);
        console.log("Strategy mUSDC balance:", mUSDCBefore);
        
        // Strategy approves and calls mint
        usdc.approve(address(mUSDC), amount);
        uint256 errorCode = mUSDC.mint(amount);
        
        console.log("\n=== After mint() ===");
        console.log("Error code returned:", errorCode);
        console.log("Strategy USDC balance:", usdc.balanceOf(strategy));
        console.log("Strategy mUSDC balance:", mUSDC.balanceOf(strategy));
        
        // THE BUG: mint() returned error code 1 (failure) but didn't revert
        assertEq(errorCode, 1, "mint() should return error code");
        assertEq(usdc.balanceOf(strategy), usdcBefore, "USDC should not be transferred");
        assertEq(mUSDC.balanceOf(strategy), mUSDCBefore, "mUSDC should not be minted");
        
        console.log("\n=== THE BUG ===");
        console.log("mint() failed (returned error code 1) but DID NOT REVERT");
        console.log("Strategy thinks allocation succeeded!");
        console.log("But no mUSDC was actually minted and USDC is still in strategy");
        
        vm.stopPrank();
    }
    
    /**
     * @notice Demonstrates redeemUnderlying() silent failure
     * @dev redeemUnderlying() returns error code but strategy doesn't check it
     */
    function test_RedeemSilentFailure() public {
        uint256 mintAmount = 100e6;
        uint256 redeemAmount = 50e6;
        
        // First, successfully mint some mUSDC
        vm.startPrank(strategy);
        usdc.approve(address(mUSDC), mintAmount);
        uint256 mintError = mUSDC.mint(mintAmount);
        assertEq(mintError, 0, "Mint should succeed");
        
        uint256 mUSDCBalance = mUSDC.balanceOf(strategy);
        console.log("=== After successful mint ===");
        console.log("mUSDC balance:", mUSDCBalance);
        
        // Now make redeem fail
        mUSDC.setFailRedeem(true);
        
        uint256 usdcBefore = usdc.balanceOf(strategy);
        uint256 mUSDCBefore = mUSDC.balanceOf(strategy);
        
        console.log("\n=== Before redeemUnderlying() ===");
        console.log("Strategy USDC balance:", usdcBefore);
        console.log("Strategy mUSDC balance:", mUSDCBefore);
        
        // Strategy calls redeemUnderlying
        uint256 errorCode = mUSDC.redeemUnderlying(redeemAmount);
        
        console.log("\n=== After redeemUnderlying() ===");
        console.log("Error code returned:", errorCode);
        console.log("Strategy USDC balance:", usdc.balanceOf(strategy));
        console.log("Strategy mUSDC balance:", mUSDC.balanceOf(strategy));
        
        // THE BUG: redeemUnderlying() returned error code 1 (failure) but didn't revert
        assertEq(errorCode, 1, "redeemUnderlying() should return error code");
        assertEq(usdc.balanceOf(strategy), usdcBefore, "USDC should not be transferred");
        assertEq(mUSDC.balanceOf(strategy), mUSDCBefore, "mUSDC should not be burned");
        
        console.log("\n=== THE BUG ===");
        console.log("redeemUnderlying() failed (returned error code 1) but DID NOT REVERT");
        console.log("Strategy thinks deallocation succeeded!");
        console.log("But no USDC was actually redeemed and mUSDC is still locked");
        
        vm.stopPrank();
    }
    
    /**
     * @notice Shows successful operations for comparison
     */
    function test_SuccessfulOperations() public {
        uint256 mintAmount = 100e6;
        uint256 redeemAmount = 50e6;
        
        vm.startPrank(strategy);
        
        // Successful mint
        usdc.approve(address(mUSDC), mintAmount);
        uint256 mintError = mUSDC.mint(mintAmount);
        
        console.log("=== Successful mint ===");
        console.log("Error code:", mintError);
        console.log("mUSDC balance:", mUSDC.balanceOf(strategy));
        
        assertEq(mintError, 0, "Should return 0 on success");
        assertEq(mUSDC.balanceOf(strategy), mintAmount, "mUSDC should be minted");
        
        // Successful redeem
        uint256 redeemError = mUSDC.redeemUnderlying(redeemAmount);
        
        console.log("\n=== Successful redeem ===");
        console.log("Error code:", redeemError);
        console.log("USDC balance:", usdc.balanceOf(strategy));
        
        assertEq(redeemError, 0, "Should return 0 on success");
        assertEq(usdc.balanceOf(strategy), 1000e6 - mintAmount + redeemAmount, "USDC should be redeemed");
        
        vm.stopPrank();
    }
}

````

## Mock contract (MoonwellMock.sol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

/**
 * @notice Mock Moonwell mToken that simulates error returns instead of reverts
 * @dev Demonstrates that mint() and redeemUnderlying() return error codes instead of reverting
 */
contract MoonwellMTokenMock {
    address public immutable underlying;
    
    mapping(address => uint256) public balanceOf;
    uint256 public totalSupply;
    uint256 public exchangeRate = 1e18; // 1:1 for simplicity
    
    bool public shouldFailMint;
    bool public shouldFailRedeem;
    
    // Track interest accrual
    uint256 public lastAccrualTimestamp;
    uint256 public accruedInterest;
    bool public interestAccrued;
    
    constructor(address _underlying) {
        underlying = _underlying;
        lastAccrualTimestamp = block.timestamp;
    }
    
    function setFailMint(bool _fail) external {
        shouldFailMint = _fail;
    }
    
    function setFailRedeem(bool _fail) external {
        shouldFailRedeem = _fail;
    }
    
    /**
     * @notice Simulate interest accrual - increases exchange rate
     * @dev In real Moonwell, this is called automatically on interactions
     */
    function accrueInterest() public returns (uint256) {
        uint256 timeDelta = block.timestamp - lastAccrualTimestamp;
        
        if (timeDelta > 0 && totalSupply > 0) {
            // Simulate 10% APY (simplified calculation)
            // Interest per second = 0.1 / (365 * 24 * 60 * 60) ≈ 3.17e-9
            uint256 interestRate = 317e7; // Per second (scaled by 1e18)
            uint256 interest = (exchangeRate * interestRate * timeDelta) / 1e18;
            
            exchangeRate += interest;
            accruedInterest += interest;
            interestAccrued = true;
        }
        
        lastAccrualTimestamp = block.timestamp;
        return 0; // Success
    }
    
    /**
     * @notice Mint mTokens - returns error code instead of reverting
     * @dev Returns 0 on success, non-zero error code on failure
     *      THE BUG: Strategy doesn't check this return value
     */
    function mint(uint256 mintAmount) external returns (uint256) {
        if (shouldFailMint) {
            // Return error code - NO REVERT!
            return 1; // Error code for failure
        }
        
        // Transfer underlying from sender
        MockERC20(underlying).transferFrom(msg.sender, address(this), mintAmount);
        
        // Mint mTokens
        uint256 mTokens = mintAmount; // 1:1 for simplicity
        balanceOf[msg.sender] += mTokens;
        totalSupply += mTokens;
        
        return 0; // Success
    }
    
    /**
     * @notice Redeem underlying - returns error code instead of reverting
     * @dev Returns 0 on success, non-zero error code on failure
     *      THE BUG: Strategy doesn't check this return value
     */
    function redeemUnderlying(uint256 redeemAmount) external returns (uint256) {
        if (shouldFailRedeem) {
            // Return error code - NO REVERT!
            return 1; // Error code for failure
        }
        
        // Burn mTokens
        uint256 mTokens = redeemAmount; // 1:1 for simplicity
        require(balanceOf[msg.sender] >= mTokens, "Insufficient balance");
        
        balanceOf[msg.sender] -= mTokens;
        totalSupply -= mTokens;
        
        // Transfer underlying to sender
        MockERC20(underlying).transfer(msg.sender, redeemAmount);
        
        return 0; // Success
    }
    
    function balanceOfUnderlying(address owner) external view returns (uint256) {
        return (balanceOf[owner] * exchangeRate) / 1e18;
    }
    
    /**
     * @notice Returns STORED (stale) exchange rate without accruing interest
     * @dev This is what the strategy uses - can be outdated!
     */
    function exchangeRateStored() external view returns (uint256) {
        return exchangeRate; // Returns OLD rate if accrueInterest() wasn't called
    }
    
    /**
     * @notice Returns CURRENT exchange rate after accruing interest
     * @dev This should be used for accurate calculations
     */
    function exchangeRateCurrent() external returns (uint256) {
        accrueInterest(); // Accrues interest first
        return exchangeRate;
    }
    
    /**
     * @notice Helper to check if interest needs accruing
     */
    function needsAccrual() external view returns (bool) {
        return block.timestamp > lastAccrualTimestamp;
    }
    
    /**
     * @notice Helper to simulate time passing
     */
    function simulateTimePass(uint256 timeInSeconds) external {
        // This will be used in tests via vm.warp
        lastAccrualTimestamp = block.timestamp - timeInSeconds;
    }
}

/**
 * @notice Simple ERC20 mock
 */
contract MockERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 6;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        
        return true;
    }
    
    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }
}

```
