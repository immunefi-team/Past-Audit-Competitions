# #46608 \[SC-Medium] Any call to the repay function can potentially be front-run by a malicious actor, lead to prevent users from repaying on time.

**Submitted on Jun 2nd 2025 at 10:25:42 UTC by @zeroK for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #46608
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `Settlement#repay` function is designed to allow a borrower to repay the debt associated with a specific loan id, preventing liquidation. However, since the function does not have affected check for who can call it, any user including a malicious actor can front-run the borrower’s intended repayment. This means an attacker or even a lender could call the repay function just before the borrower does, effectively preventing the borrower from repaying in time and cause lose of gas for the borrower(griefing) In some cases, this could lead to the borrower’s position being liquidated despite their intention to repay.

## Vulnerability Details

the function repay implemented as below:

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

We can see that the `Settlement#repay` function allows any caller to set all the required inputs, regardless of who they are. This can be useful because it means anyone can repay the borrower’s debtsuch as a borrower’s second address or even third parties.

However, the function has an issue when the borrower themselves intends to repay all of their debt. This is because of how the `loanInfo.repay` function is implemented. Let’s take a closer look at how this function works:

```solidity
    function repay(LoanInfo memory loanInfo, uint256 repayAmt) internal pure {
        if (loanInfo.debtData.debtAmt < repayAmt) {
            revert SettlementErrors.RepayAmountExceedsDebtAmt(loanInfo.debtData.debtAmt, repayAmt);
        }
        loanInfo.debtData.debtAmt -= repayAmt;
    }


```

we can see that the check `if (loanInfo.debtData.debtAmt < repayAmt)` will revert if the loand.debtAmt is smaller than the repayAmt input, this can lead to scenario below to be possible:

* bob want to repay all of his debt back, which is 100 USDC.
* alice(can be the lender too to get its value back when transfer invoked) recognize bob transaction and front run his call by repay back 1 usdc and setting all required input with setting the `removeCollateralAmt == 0` to avoid checking for caller is borrrower .
* this way alice prevented bob to pay back all of his debt amount because the check in `if (loanInfo.debtData.debtAmt < repayAmt)` which equal to `99 < 100`.

the issue exist in how the loanInfo.repay function work which it should set the repay amount to loanInfo.debtData.debtAmt when its bigger than the debt amount info.

## Impact Details

malicious user can front-run honest borrower who intend to pay their debt fully and cause damage to them and in certain cases can lead to liquidation.

## Recommend

a simple fix can be implemented as below:

```solidity
    function repay(LoanInfo memory loanInfo, uint256 repayAmt) internal pure {
        if (loanInfo.debtData.debtAmt < repayAmt) {
            repayAmt = loanInfo.debtData.debtAmt; //@audit since 100 is bigger than the debt amount, set repay amount to 99 and transfer back the 1 back to caller or borrower in repay function
        }

        loanInfo.debtData.debtAmt -= repayAmt;
    }



```

## Proof of Concept

## Proof of Concept

* use the settlement.t.sol to run the test
* use the testRepay function which itself is ready to simulate full repay action
* in the testRepay the debt amount is 1000e6 and the function repay back 100e6 only.
* add a call to the repay function and repay 1e6 amount and then invoke another call to repay 1000e6, this is to simulate a front running on chain which the attacker front run the victim.
* this attack only affected the contracts that get deployed on ethereum which can be supported network according to the scripts here: <https://github.com/term-structure/tsi-contract/blob/48bc11e2ce9febab858785d79ad59f8df667a9aa/script/deploy/DeploySettlement.s.sol#L27-L39>
