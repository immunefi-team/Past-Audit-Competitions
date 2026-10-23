# #47124 \[SC-Insight] Minimum Debt Value Updates Trigger Instant Liquidation Condition Changes

**Submitted on Jun 9th 2025 at 03:53:09 UTC by @Catchme for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47124
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Protocol insolvency
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The Settlement contract allows the owner to instantly change the minimum debt value threshold without any timelock protection. This parameter directly affects liquidation conditions, potentially forcing immediate full liquidations on previously healthy loans. This creates a significant risk for borrowers who may have their collateral fully liquidated without warning or time to react when this parameter is increased.

## Vulnerability Details

The vulnerability exists in the `setMinimumDebtValue()` function at src/Settlement.sol:74-77:

```solidity
function setMinimumDebtValue(uint256 minimumDebtValue_) external onlyOwner {
    _minimumDebtValue = minimumDebtValue_;
    emit SetMinimumDebtValue(minimumDebtValue_);
}
```

This function allows the contract owner to immediately update the global minimum debt value without any delay or transition period. The `_minimumDebtValue` parameter is critical as it directly affects liquidation logic in `LoanLib.liquidaitonInfo()`:

```solidity
// src/lib/LoanLib.sol:103-115
if (ltv >= loanInfo.debtData.lltv) {
    liquidateable = true;
    maxLiquidationAmt = _calculateMaxLiquidationAmt(loanInfo, collateralPrice, debtPrice);
    uint256 remainningDebtAmt = loanInfo.debtData.debtAmt - maxLiquidationAmt;
    if (remainningDebtAmt > 0 && remainningDebtAmt < _calculateMinimumDebtAmt(minimumDebtValue, debtPrice))
    {
        maxLiquidationAmt = loanInfo.debtData.debtAmt;  // Forces full liquidation
    }
}
```

The logical flaw is that increasing `_minimumDebtValue` can instantly transform loans that were only partially liquidatable into fully liquidatable positions. This creates a sudden change in liquidation risk without providing borrowers any notice or time to adjust their positions.

This contrasts with other critical parameter changes in the contract (like operator updates) that use proper timelock protection:

```solidity
function submitPendingOperator(address _newOperator) external onlyOwner {
    _pendingOperator.update(_newOperator, Constants.TIMELOCK);
    emit PendingOperatorSet(_newOperator);
}

function acceptOperator() external afterTimelock(_pendingOperator.validAt) {
    // Implementation with timelock protection
}
```

## Impact Details

1. **Sudden Liquidation Risk**: Borrowers with healthy loans that would normally qualify for partial liquidation may suddenly face full liquidation without warning
2. **Cascading Liquidations**: A significant increase in the minimum debt value could trigger simultaneous full liquidations across multiple positions, potentially causing market instability

## Proof of Concept

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import "forge-std/Test.sol";
import "../src/Settlement.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        _mint(msg.sender, 1000000 * 10**18);
    }
    
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract MockOracle is IOracle {
    // Allow price manipulation for testing
    mapping(address => uint256) private tokenPrices;
    
    constructor() {
        // Default prices
        tokenPrices[address(0)] = 1e18; // Default price
    }
    
    function setTokenPrice(address token, uint256 price) external {
        tokenPrices[token] = price;
    }
    
    function getPrice(address token) external view override returns (PriceInfo memory) {
        uint256 price = tokenPrices[token];
        if (price == 0) price = tokenPrices[address(0)]; // Default price
        
        return PriceInfo({
            price: price,
            timestamp: block.timestamp,
            source: "MOCK"
        });
    }
}

contract MinimumDebtValuePOC is Test {
    Settlement settlement;
    MockERC20 collateralToken;
    MockERC20 debtToken;
    MockOracle oracle;
    
    address admin = address(1);
    address feeCollector = address(2);
    address operator = address(3);
    address lender = address(4);
    address borrower = address(5);
    address liquidator = address(6);
    
    uint256 initialCollateral = 100 * 10**18;  // 100 tokens
    uint256 initialBorrow = 70 * 10**18;      // 70 tokens
    uint256 initialMinDebtValue = 10 * 10**18; // $10 minimum debt value
    
    string loanIdStr = "test-loan";
    bytes32 loanId;
    
    function setUp() public {
        // Deploy tokens
        collateralToken = new MockERC20("Collateral Token", "COL");
        debtToken = new MockERC20("Debt Token", "DEBT");
        
        // Deploy oracle
        oracle = new MockOracle();
        
        // Set token prices in oracle (1:1 for simplicity)
        oracle.setTokenPrice(address(collateralToken), 1 * 10**18);
        oracle.setTokenPrice(address(debtToken), 1 * 10**18);
        
        // Deploy settlement contract with initial minimum debt value
        settlement = new Settlement(admin, feeCollector, operator, address(oracle));
        vm.startPrank(admin);
        settlement.setMinimumDebtValue(initialMinDebtValue);
        vm.stopPrank();
        
        // Distribute tokens
        collateralToken.transfer(borrower, 200 * 10**18);
        debtToken.transfer(lender, 200 * 10**18);
        debtToken.transfer(liquidator, 200 * 10**18);
        
        // Set up approvals
        vm.startPrank(borrower);
        collateralToken.approve(address(settlement), type(uint256).max);
        debtToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(lender);
        debtToken.approve(address(settlement), type(uint256).max);
        collateralToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(liquidator);
        debtToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        // Create and settle a loan
        loanId = bytes32(bytes(loanIdStr));
        createAndSettleLoan();
    }
    
    function createAndSettleLoan() internal {
        string memory settlementIdStr = "test-settlement";
        
        // Create a loan with 100 tokens collateral and 70 tokens debt (70% LTV)
        string[] memory loanIds = new string[](1);
        loanIds[0] = loanIdStr;
        
        LoanInfo[] memory loanInfos = new LoanInfo[](1);
        DebtData memory debtData = DebtData({
            borrowedAmt: initialBorrow,
            debtAmt: initialBorrow,
            feeAmt: 0,
            collateralAmt: initialCollateral
        });
        
        loanInfos[0] = LoanInfo({
            maker: lender,
            borrower: borrower,
            lender: lender,
            debtTokenAddr: address(debtToken),
            collateralTokenAddr: address(collateralToken),
            settlementId: bytes32(0),
            debtData: debtData,
            settled: false
        });
        
        SettleInfo memory settleInfo = SettleInfo({
            taker: borrower,
            takerType: TakerType.BORROW,
            expiryTime: block.timestamp + 1 days
        });
        
        // Mock signature
        bytes memory signature = abi.encodePacked(bytes32(0), bytes32(0), uint8(0));
        vm.mockCall(
            address(operator),
            abi.encodeWithSelector(SignatureChecker.isValidSignatureNow.selector),
            abi.encode(true)
        );
        
        // Create settlement as borrower
        vm.startPrank(borrower);
        settlement.createSettlement(settlementIdStr, settleInfo, loanIds, loanInfos, signature);
        vm.stopPrank();
        
        // Settle loan as lender
        vm.startPrank(lender);
        settlement.settle(loanIdStr);
        vm.stopPrank();
    }
    
    function testMinimumDebtValueImpact() public {
        // Simulate a price drop to put the loan in a situation where partial liquidation is possible
        // Let's say collateral value drops from 100 to 80 (20% drop)
        oracle.setTokenPrice(address(collateralToken), 0.8 * 10**18);
        
        console.log("===== Initial State =====");
        console.log("Initial minimum debt value: $", initialMinDebtValue / 1e18);
        
        // Check initial liquidation status
        (bool initialLiquidatable, , uint256 initialMaxLiquidationAmt, , ) = settlement.liquidationInfo(loanIdStr);
        
        console.log("Loan liquidatable: ", initialLiquidatable ? "Yes" : "No");
        console.log("Maximum liquidation amount: ", initialMaxLiquidationAmt / 1e18);
        
        // Calculate expected remaining debt after partial liquidation
        uint256 expectedRemainingDebt = initialBorrow - initialMaxLiquidationAmt;
        console.log("Expected remaining debt after partial liquidation: ", expectedRemainingDebt / 1e18);
        
        // Verify this is above the minimum debt threshold (doesn't force full liquidation)
        bool isPartialLiquidation = expectedRemainingDebt >= initialMinDebtValue;
        console.log("Is partial liquidation possible: ", isPartialLiquidation ? "Yes" : "No");
        
        // Now let's increase the minimum debt value significantly
        uint256 newMinDebtValue = 30 * 10**18; // $30 minimum debt
        
        console.log("\n===== Updating Minimum Debt Value =====");
        console.log("New minimum debt value: $", newMinDebtValue / 1e18);
        
        vm.startPrank(admin);
        settlement.setMinimumDebtValue(newMinDebtValue);
        vm.stopPrank();
        
        // Check liquidation status after minimum debt value change
        (bool newLiquidatable, , uint256 newMaxLiquidationAmt, , ) = settlement.liquidationInfo(loanIdStr);
        
        console.log("\n===== After Minimum Debt Value Update =====");
        console.log("Loan liquidatable: ", newLiquidatable ? "Yes" : "No");
        console.log("Maximum liquidation amount: ", newMaxLiquidationAmt / 1e18);
        
        // Verify if this now forces full liquidation because remaining debt would be below minimum
        bool isFullLiquidation = newMaxLiquidationAmt == initialBorrow;
        console.log("Is full liquidation forced: ", isFullLiquidation ? "Yes" : "No");
        
        // Check that the remaining debt would be below minimum threshold (causes full liquidation)
        console.log("Original expected remaining debt: ", expectedRemainingDebt / 1e18);
        console.log("New minimum debt value: ", newMinDebtValue / 1e18);
        console.log("Remaining debt below new minimum: ", (expectedRemainingDebt < newMinDebtValue) ? "Yes" : "No");
        
        console.log("\n===== Demonstrating Impact on Borrower =====");
        console.log("1. Original liquidation would leave borrower with partial position");
        console.log("2. After parameter change, borrower faces full liquidation");
        console.log("3. Immediate change with no warning to adjust position");
        
        // Execute liquidation to show the full impact
        vm.startPrank(liquidator);
        uint256 liquidationAmount = 20 * 10**18; // Try to liquidate just a portion
        console.log("\n===== Attempting Partial Liquidation =====");
        console.log("Liquidator requested amount: ", liquidationAmount / 1e18);
        
        // Record balances before liquidation
        uint256 borrowerColBefore = collateralToken.balanceOf(borrower);
        uint256 lenderDebtBefore = debtToken.balanceOf(lender);
        
        // This will force full liquidation despite requesting partial amount
        settlement.liquidate(loanIdStr, liquidationAmount);
        vm.stopPrank();
        
        // Check if loan was fully liquidated (would be deleted if fully liquidated)
        LoanInfo memory loanInfo = settlement.getLoan(loanIdStr);
        bool loanDeleted = loanInfo.maker == address(0);
        
        console.log("Loan fully liquidated: ", loanDeleted ? "Yes" : "No");
        
        // If we can't check the loan directly because it was deleted, show other evidence
        uint256 borrowerColAfter = collateralToken.balanceOf(borrower);
        uint256 lenderDebtAfter = debtToken.balanceOf(lender);
        
        console.log("\n===== Financial Impact Evidence =====");
        console.log("Borrower collateral returned: ", (borrowerColAfter - borrowerColBefore) / 1e18);
        console.log("Lender debt received: ", (lenderDebtAfter - lenderDebtBefore) / 1e18);
    }
}
```
