# #46903 \[SC-Critical] malicious borrower can take theft of other borrower collateral

**Submitted on Jun 6th 2025 at 04:35:15 UTC by @zeroK for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #46903
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The createSettlement function allows a single lender to create loan positions for multiple borrowers, each with different debt, borrowing amounts, and collateral amounts. Additionally, there are functions that enable either the borrower or the lender to modify these loan positions, such as adding, removing, or repaying collateral or debt. However, many of these functions are callable even after the settlement has expired or before the loan has been marked as settled. A malicious borrower could exploit this design to withdraw other borrowers’ collateral from the lender by leveraging the `addCollateralBeforeSettle` and `repay` functions.

## Vulnerability Details

the function createSettlement implemented as below, in our case taker type == lend:

```solidity
function createSettlement(
        string memory _settlementId,
        SettleInfo calldata _settleInfo,
        string[] memory _loanIds,
        LoanInfo[] calldata _loans,
        bytes calldata _signature
    ) external nonReentrant {
        // // Verify signature
        bytes32 digest = _getSettlementHash(_settlementId, _settleInfo, _loanIds, _loans);
        if (!SignatureChecker.isValidSignatureNow(_operator, digest, _signature)) {
            revert InvalidSignature();
        }
        bytes32 settlementId = _settlementId.toBytes32();
        if (settlements[settlementId].taker != address(0)) {
            revert SettlementAlreadyExists(settlementId);
        }
        if (_settleInfo.expiryTime < block.timestamp) {
            revert SettlementExpired(block.timestamp, _settleInfo.expiryTime);
        }
        if (_settleInfo.taker != msg.sender) {
            revert TakerNotMatched(settlementId, _settleInfo.taker, msg.sender);
        }
        if (_loans.length == 0) {
            revert EmptyLoan();
        }
        uint256 totalCollateralAmt;
        uint256 totalDebtAmt;
        address debtToken = _loans[0].debtTokenAddr;
        address collateralToken = _loans[0].collateralTokenAddr;
        for (uint256 i = 0; i < _loanIds.length; i++) {
            bytes32 loanId = _loanIds[i].toBytes32();
            LoanInfo memory loan = _loans[i];
            if (loans[loanId].maker != address(0)) {
                revert LoanAlreadyExisted(loanId);
            }
            loan.settlementId = settlementId;
            loan.settled = false;
            loans[loanId] = loan;
            totalCollateralAmt += loan.debtData.collateralAmt;
            totalDebtAmt += loan.debtData.debtAmt;
            if (_settleInfo.takerType == TakerType.BORROW && loan.collateralTokenAddr != collateralToken) {
                revert TakerNotMatched(loanId, collateralToken, loan.collateralTokenAddr);
            } else if (_settleInfo.takerType == TakerType.LEND && loan.debtTokenAddr != debtToken) {
                revert TakerNotMatched(loanId, debtToken, loan.debtTokenAddr);
            }
        }
        if (_settleInfo.takerType == TakerType.LEND) {
            uint256 allowance = IERC20(collateralToken).allowance(msg.sender, address(this));
            if (allowance < Constants.ALLOWANCE_SCALE * totalCollateralAmt) {
                revert TokenAllowanceInsufficient(allowance, Constants.ALLOWANCE_SCALE * totalCollateralAmt);
            }
            allowance = IERC20(debtToken).allowance(msg.sender, address(this));
            if (allowance < totalDebtAmt) {
                revert TokenAllowanceInsufficient(allowance, totalDebtAmt);
            }
        } else {
            uint256 allowance = IERC20(collateralToken).allowance(msg.sender, address(this));
            if (allowance < totalCollateralAmt) {
                revert TokenAllowanceInsufficient(allowance, totalCollateralAmt);
            }
        }

        settlements[settlementId] = _settleInfo;

        emit SettlementCreated(msg.sender, settlementId);
    }



```

The lender (who is the msg.sender) is required to approve the Settlement contract to spend an amount equal to 10 times the total collateral received from all borrowers. This requirement ensures that the liquidation process can be executed smoothly when needed. For example, if the total collateral from 5 loan positions is 5,000 ETH (where each borrower provides 1,000 ETH as collateral to receive 1.9 million USDC), the lender must approve the Settlement contract to spend 10 times that amount (e.g 50,000 ETH)

```solidity
         uint256 allowance = IERC20(collateralToken).allowance(msg.sender, address(this));
            if (allowance < Constants.ALLOWANCE_SCALE * totalCollateralAmt) {
                revert TokenAllowanceInsufficient(allowance, Constants.ALLOWANCE_SCALE * totalCollateralAmt);
         

```

Imagine that there are 6 loans created 5 honest users(borrower), and 1 malicious borrower. Let’s say the createSettlement function was invoked correctly, and everything initially worked as expected. Now, the malicious borrower can exploit this system to steal collateral from the lender that originally came from honest borrowers.

* The malicious borrower can simply wait until the loan position expires (settlement expires). Meanwhile, the other honest borrowers would execute calls to the settle function, transferring their collateral to the lender’s wallet.
* the borrower invokes the `addCollateralBeforeSettle` function, and increase the collateral amount to equal to 5k eth which is same collateral value that borrower transfer to lender in settle function call(borrower can increase this value up to all collateral lender hold and gave approve to settlement.sol contract):

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
            uint256 allowance = IERC20(loanInfo.collateralTokenAddr).allowance(msg.sender, address(this));
            if (allowance < loanInfo.debtData.collateralAmt) {
                revert TokenAllowanceInsufficient(allowance, loanInfo.debtData.collateralAmt);
            }
        }

        loans[loanId] = loanInfo;

        emit CollateralAdded(msg.sender, loanId, addedAmount);
    }

```

* we can see the addCollateralBeforeSettle is callable even if the settlement is expired(loan expired) and it increase the collateral debt without transfer the added collateral.
* After that, the borrower can call the repay function. What this function does is transfer the debt amount of the loan 1.9 million USDC in this case, to the lender, and then transfer the collateral back to the borrower. The collateral should be 1,000 ETH, and it should only be transferred after the settlement is executed. However, in this scenario, the borrower would actually receive 5,000 ETH, which the lender got from other borrowers, This is possible due to a few reasons: A) The lender has already approved the contract to spend collateral on their behalf.

B) The borrower increased the loan’s collateral or debt, and although the loan expired, it can still be modified and repaid.

C) The health check will pass because the collateral amount was increased

This allows the malicious user to effectively steal the lender’s collateral and also impact the liquidation process, creating a serious vulnerability.

## Impact Details

due to lack of check in addCollateralBeforeSettle and repay functions, malicious user can take theft of the lender collateral.

## Recommend

It’s better to prevent any calls to the repay function when settled == false or when the loan has expired. This is because the addCollateralBeforeSettle function can be invoked before settled becomes true, allowing an attacker to increase the collateral amount of a position. If this happens, the attacker should then invoke the settle function, which actually takes the collateral from them. Otherwise, the repay function will revert because of the added settled == true check (the same check can also be applied to the liquidation process).

Regarding the removeCollateral function, although it’s possible to call it, it would revert in most cases since it checks the position’s health. When the collateral is zero, the ltv typically exceeds the mltv (maximum Liquidation Threshold) under normal conditions, preventing collateral removal.

## Proof of Concept

## Proof of Concept

note: run all the steps in one test function in settlement.t.sol for better experience:

* first create a settlement by invoking the createSettlement function, create 6 loan positions, total of all of them should be equal to 6k eth(1k for each) and the debt should be around 1.7M USDC so its not liquidated(1.9M USDC can be set but LTV and MLTV should be managed)
* then invoke settlement for all 5 loan expect one of them, which is the hacker loan, this can be reached by invoke call to settlement function for each loan id.
* then the hacker loan maker should call addCollateralBeforeSettle to set the debtAmount == 5k, and then invoke the repay to get all lender collateral that got from borrower(you can wait to the settlement to be expired, but this not matter since there is no checks in any mentioned function for expiry time)
