# 58192 sc high tokeautoeth strategy tokens locked when autopool router enforces maxdeposit cap

**Submitted on Oct 31st 2025 at 09:43:06 UTC by @T0nraq for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58192
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

The `TokeAutoEth` strategy's use of `router.depositMax()` causes WETH tokens to become permanently locked in the strategy contract when `router::maxDeposit` limits are enforced. The VaultV2 transfers the full requested `amount` to the strategy, but `depositMax()` only deposits `min(balance, maxDeposit)`, leaving the difference trapped in the contract with no recovery mechanism.

## Vulnerable Code

### VaultV2 Transfers Full Amount to Strategy

**File**: `lib/vault-v2/src/VaultV2.sol`

```solidity
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        accrueInterest();

        SafeERC20Lib.safeTransfer(asset, adapter, assets); // <<<<<@ full amt sent
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            _caps.allocation = (int256(_caps.allocation) + change).toUint256();

            require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());
            require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());
            require(
                _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
                ErrorsLib.RelativeCapExceeded()
            );
        }
        emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
    }
```

### Strategy Allocation (Fails to Deploy Full Amount)

**File**: `src/strategies/mainnet/TokeAutoEth.sol`\
**Lines**: 56-63

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
    
    // Approves full amount received from vault
    TokenUtils.safeApprove(address(weth), address(router), amount);
    
    // But depositMax() may deposit LESS than amount due to maxDeposit cap
    uint256 shares = router.depositMax(autoEth, address(this), 0);
    
    TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    
    // Returns amount, claiming full deployment even if partial
    return amount;
}
```

### Router DepositMax (Caps at maxDeposit)

**Contract**: `AutopilotRouter` ([0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21](https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21))

```solidity
function depositMax(
    IAutopool vault,
    address to,
    uint256 minSharesOut
) public payable override returns (uint256 sharesOut) {
    IERC20 asset = IERC20(vault.asset());
    uint256 assetBalance = asset.balanceOf(msg.sender);
    
    // Gets vault's maximum deposit limit
    uint256 maxDeposit = vault.maxDeposit(to);
    
    // Takes MINIMUM of balance and maxDeposit
    // If maxDeposit < balance, excess tokens are left behind
    uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
    
    pullToken(asset, amount, address(this));
    approve(IERC20(vault.asset()), address(vault), amount);
    return deposit(vault, to, amount, minSharesOut);
}
```

## Vulnerability Details

### The Token Lock Flow

**Step 1: VaultV2 transfers full amount to strategy**

```solidity
// Allocator requests allocation of 1000 WETH
_allocate(tokeAutoEthStrategy, 1000e18);

// VaultV2 transfers ENTIRE 1000 WETH to strategy
IERC20(WETH).safeTransfer(tokeAutoEthStrategy, 1000e18);
```

**Step 2: Strategy approves and calls depositMax**

```solidity
// Strategy has 1000 WETH balance
TokenUtils.safeApprove(address(weth), address(router), 1000e18); // Approves 1000 WETH

// Calls router.depositMax()
uint256 shares = router.depositMax(autoEth, address(this), 0);
```

**Step 3: Router caps deposit at maxDeposit limit**

```solidity
// Router checks vault limits
uint256 assetBalance = asset.balanceOf(msg.sender);  // Returns 1000 WETH
uint256 maxDeposit = vault.maxDeposit(to);           // Returns 900 WETH (vault is near capacity)

// Takes MINIMUM - only deposits 900 WETH
uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance; // amount = 900 WETH

// Deposits only 900 WETH to vault
pullToken(asset, 900e18, address(this));
```

**Step 4: Excess tokens are locked**

```solidity
// Strategy balance after deposit: 1000 - 900 = 100 WETH
// These 100 WETH are PERMANENTLY LOCKED:
// - No mechanism to return them to VaultV2
// - No recovery function exists
// - Cannot be reallocated or withdrawn
```

### Root Cause Analysis

The vulnerability arises from a **three-way contract interaction failure**:

1. **VaultV2 Assumption**: Transfers full `amount` expecting complete deployment
2. **Router Reality**: Respects `maxDeposit` limits and may deposit less
3. **Strategy Gap**: No logic to handle partial deposits or return excess funds

The strategy's `_allocate()` function returns `amount` (claiming full deployment) even when only a portion was actually deposited, creating an accounting mismatch that traps the difference.

## Impact

### Severity Justification

* **Permanent Capital Loss**: Tokens become irrecoverably stuck in strategy

## Recommended Fix

### Direct Vault Deposit with Limit Enforcement (Recommended)

Replace `router.depositMax()` with direct vault deposit and enforce limits upfront:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
    
    // Check maxDeposit BEFORE attempting allocation
    uint256 maxDeposit = autoEth.maxDeposit(address(this));
    require(amount <= maxDeposit, "Amount exceeds vault deposit limit");
    
    // Approve and deposit exact amount
    TokenUtils.safeApprove(address(weth), address(autoEth), amount);
    uint256 shares = autoEth.deposit(amount, address(this));
    
    TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

## References

* **POC Test**: `test/TokeAutoEthBugs.t.sol::test_TokenLockWhenMaxDepositExceeded`
* **Vulnerable Code**: `src/strategies/mainnet/TokeAutoEth.sol:66`
* **VaultV2 Transfer**: `lib/vault-v2/src/VaultV2.sol` (allocate function)
* **Router Contract**: [AutopilotRouter 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21](https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21)

## Proof of Concept

## Proof of Concept

### Steps to run POC

1. Create directory v3-poc/test/TokeAutoEthBugs.t.sol
2. Paste the snippet below in the newly created file and run with `forge test`

```soldity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console.sol";

/**
 * @title TokeAutoEthBugsTest
 * @notice This test demonstrates two critical bugs in the TokeAutoEthStrategy._allocate() function
 * 
 * BUG 1: max balance deposit.
 * - router.depositMax() deposits the ENTIRE balance of msg.sender, not just the approved amount
 * - An attacker can frontrun and send 1 wei of WETH to the strategy contract
 * - This causes router to try depositing more than the approved amount, resulting in a revert
 * 
 * BUG 2: Token Lock/Loss
 * - If the strategy contract holds more WETH than vault.maxDeposit(to) allows
 * - router.depositMax() will only deposit up to maxDeposit amount
 * - The remaining WETH gets stuck in the strategy contract with no recovery mechanism
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
     * @notice BUG 1: Demonstrates frontrun DOS attack
     * @dev An attacker sends 1 wei of WETH to the strategy, causing allocation to fail
     */
    // function test_Bug1_maxBalanaceUsage() public {
    //     uint256 amountToAllocate = 10 ether;
        
    //     // Initial balance check
    //     uint256 initialBalance = weth.balanceOf(address(strategy));
    //     assertEq(initialBalance, 100 ether);
        
    //     // ATTACKER FRONTRUNS: Sends 1 wei of WETH to strategy
    //     vm.prank(attacker);
    //     weth.mint(address(strategy), 1);
        
    //     uint256 balanceAfterAttack = weth.balanceOf(address(strategy));
    //     assertEq(balanceAfterAttack, 100 ether + 1);
        
    //     // Now when strategy tries to allocate, it will fail
    //     // The strategy approves only 'amount' but router tries to deposit entire balance
    //     vm.expectRevert("ERC20: insufficient allowance");
    //     strategy.allocate(amountToAllocate);
        
    //     console.log("BUG 1 DEMONSTRATED:");
    //     console.log("- Strategy approved:", amountToAllocate);
    //     console.log("- Strategy balance:", balanceAfterAttack);
    //     console.log("- Router tried to deposit:", balanceAfterAttack);
    //     console.log("- Result: REVERT - Complete DOS");
    // }
    
    /**
     * @notice BUG 2: Demonstrates token lock when balance exceeds maxDeposit
     * @dev When strategy has more WETH than vault's maxDeposit, excess tokens get locked
     */
    function test_Bug2_TokenLock() public {
        // Set vault's maxDeposit to a lower amount
        uint256 maxDeposit = 50 ether;
        autoEth.setMaxDeposit(maxDeposit);
        
        uint256 amountToAllocate = 100 ether; // More than maxDeposit
        uint256 initialBalance = weth.balanceOf(address(strategy));
        
        assertEq(initialBalance, 100 ether);
        assertEq(autoEth.maxDeposit(address(strategy)), maxDeposit);
        
        // Strategy approves full amount and calls router.depositMax
        strategy.allocate(amountToAllocate);
        
        // Check what happened
        uint256 finalBalance = weth.balanceOf(address(strategy));
        uint256 deposited = initialBalance - finalBalance;
        uint256 locked = finalBalance;
        
        console.log("\nBUG 2 DEMONSTRATED:");
        console.log("- Initial balance:", initialBalance);
        console.log("- Approved amount:", amountToAllocate);
        console.log("- Max deposit allowed:", maxDeposit);
        console.log("- Actually deposited:", deposited);
        console.log("- Tokens LOCKED in contract:", locked);
        
        // Assert that tokens are locked
        assertEq(deposited, maxDeposit, "Only maxDeposit amount was deposited");
        assertEq(locked, 50 ether, "50 ether is now locked in the strategy");
        assertGt(locked, 0, "Tokens are stuck in strategy with no recovery");
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
