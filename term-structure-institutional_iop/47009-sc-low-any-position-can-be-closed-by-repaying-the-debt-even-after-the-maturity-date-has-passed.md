# #47009 \[SC-Low] Any position can be closed (by repaying the debt) even after the maturity date has passed

**Submitted on Jun 7th 2025 at 19:51:39 UTC by @zeroK for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47009
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Block stuffing

## Description

## Brief/Intro

Any loan created has a maturity date, which is important because it enforces a repayment deadline. If the borrower does not repay on time, the position should be liquidated to ensure that the lender and the protocol are compensated. However, currently there is no restriction preventing repayment after the maturity date. Borrowers can repay or modify their position at any time, even after maturity. This undermines the liquidation process because users can front-run any attempt at liquidation by repaying their debt after maturity, effectively blocking liquidation for that loan position.

position should be liquidated right when the maturity reached for position: <https://docs.institutional.ts.finance/features/liquidation-and-physical-delivery#liquidation-ltv-lltv>

## Vulnerability Details

we can see when a loan created, the maturity time should be set as input parameter:

```solidity


    function createSettlement(
        string memory _settlementId,
        SettleInfo calldata _settleInfo,
        string[] memory _loanIds,
        LoanInfo[] calldata _loans,
        bytes calldata _signature
    ) external nonReentrant {

.....


// storage

struct LoanInfo {
    bytes32 settlementId; // ID of the settlement this loan belongs to
    address maker; // who created the loan
    address lender; // who provides the funds
    address borrower; // who receives the funds(borrow)
    address collateralTokenAddr; // address of the collateral token
    address debtTokenAddr; // address of the debt token
    DebtData debtData; 
    bool settled; // has the loan been settled
}
struct DebtData {
    uint256 collateralAmt; // amount of the collateral
    uint256 debtAmt; // debt amount
    uint256 borrowedAmt; // how muhch borrowed
    uint256 feeAmt; // fee to pay to the protocol
    uint64 maturity; // when the loan matures
    uint32 lltv; // loan to value liquidation ratio in basis point
    uint32 mltv; // maximum loan to value
}


```

however, there is nothing prevent the borrower to invoke the repay function, even when the maturity passed, since there is not check to prevent invoking the function when timestamp > maturity:

```solidity

 function repay(string memory _loanId, uint256 repayAmt, uint256 removeCollateralAmt) external nonReentrant {
        bytes32 loanId = _loanId.toBytes32();
        LoanInfo memory loanInfo = loans[loanId];

        loanInfo.repay(repayAmt);
        if (removeCollateralAmt > 0) {
            loanInfo.checkBorrower(loanId, msg.sender);
            loanInfo.removeCollateral(removeCollateralAmt, _oracle);
        }

        IERC20(loanInfo.debtTokenAddr).safeTransferFrom(msg.sender, loanInfo.lender, repayAmt);
        if (removeCollateralAmt > 0) {
            IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(
                loanInfo.lender, loanInfo.borrower, removeCollateralAmt
            );
        }

        if (loanInfo.debtData.debtAmt == 0 && loanInfo.debtData.collateralAmt == 0) {
            delete loans[loanId];
        } else {
            loans[loanId] = loanInfo;
        }
        emit Repaid(loanId, repayAmt, removeCollateralAmt);
    }

```

this can lead calls to liquidation for mature position to fail since the loan deleted:

```solidity

    function liquidate(string memory _loanId, uint256 liquidationAmt) external nonReentrant {
        bytes32 loanId = _loanId.toBytes32();
        LoanInfo memory loanInfo = loans[loanId]; //@audit not exist
        (uint256 collateralToLiquidator, uint256 collateralToProtocol) =
            loanInfo.liquidate(loanId, _oracle, _minimumDebtValue, liquidationAmt);



```

this can block the liquidation process invoked by the bot, it can affect badly when batch of liquidation executed in one transaction.

## Impact Details

maturity of the loans not respected in repay function which lead to prevent liquidations.

## Recommend

add check for maturity in repay function, prevent repaying when the position passed the maturity.

## Proof of Concept

## Proof of Concept

* first thing create a settlement.
* set maturity to block.timestamp + 10 days.
* invoke the repay function, before that, wrap block.timestamp \`wrap(block.timestamp +10 days)
* we can see the loan repaid and deleted even it get matured.
