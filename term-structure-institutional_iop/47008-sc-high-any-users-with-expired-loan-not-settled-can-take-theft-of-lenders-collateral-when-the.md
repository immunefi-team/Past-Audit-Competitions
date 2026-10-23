# #47008 \[SC-High] any users with expired loan(not settled) can take theft of lenders collateral when the collateral price increase

**Submitted on Jun 7th 2025 at 19:25:27 UTC by @zeroK for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47008
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The removeCollateral function is designed to allow the borrower to reduce their collateral in a loan position, typically when the collateral value increases and exceeds the debt amount. For example, if the collateral is initially 100 ETH (worth 2 million USDC) against a debt of 1.9 million USDC, and the ETH price rises to $3,000 per ETH (making the collateral worth 3 million USDC), the borrower can remove excess collateral to bring it closer to the required 2 million USDC. However, a malicious borrower could exploit this feature to steal the lender’s collateral. This is possible because the removeCollateral function allows calls even for expired or non settled positions. If the position is not settled, the collateral has not yet been transferred by borrower to lender, and the borrower can still call removeCollateral to withdraw free collateral from the lender’s wallet, since the lender had already approved the settlement contract to manage the collateral on their behalf.

## Vulnerability Details

When a loan position is created, it can either be a lend position or a borrow position. In our case, it’s a lend position, which means the caller of the createSettlement function is the lender (the taker).

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

When the position type is set to lend, the lender is required to grant approval to the settlement contract to transfer collateral on their behalf. After that, the lender must wait for the borrower to invoke the settle function to complete the settlement process. However, a borrower can bypass this intended flow by skipping the settle function entirely and instead invoking the removeCollateral function. this exploit works as follows: the borrower waits for the collateral price to increase significantly (for example, from $2,000 to $3,000 per ETH), then calls `removeCollateral` to withdraw collateral, the removeCollateral function does not check whether the position is settled or expired meaning it can be invoked even on expired or non settled positions. As a result, the borrower can withdraw the collateral without repaying any debt, stealing the lender’s funds.

```solidity
    function removeCollateral(string memory _loanId, uint256 removeCollateralAmt) external nonReentrant {
        bytes32 loanId = _loanId.toBytes32();
        LoanInfo memory loanInfo = loans[loanId];
        loanInfo.checkBorrower(loanId, msg.sender);
        loanInfo.removeCollateral(removeCollateralAmt, _oracle);

        loans[loanId] = loanInfo;

        IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(loanInfo.lender, loanInfo.borrower, removeCollateralAmt);

        emit CollateralRemoved(loanId, removeCollateralAmt);
    }

    function removeCollateral(LoanInfo memory loanInfo, uint256 removingCollateralAmt, IOracle oracle) internal view {
        if (loanInfo.debtData.collateralAmt < removingCollateralAmt) {
            revert SettlementErrors.CollateralRemovedAmtExceedsCollateralAmt(
                loanInfo.debtData.collateralAmt, removingCollateralAmt
            );
        }
        loanInfo.debtData.collateralAmt -= removingCollateralAmt;
        checkHealth(loanInfo, oracle);
    }


```

## Impact Details

Any borrower can effectively steal yield or collateral from any position created where the maker is the user (borrower). By waiting for the collateral price to rise, the borrower can invoke functions like removeCollateral even on expired or non settled positions, thereby extracting collateral without repaying the debt. This allows a malicious borrower to profit from the increased collateral value and siphon funds from the lender’s address.

## Recommend

It is recommended to add a check to prevent functions like removeCollateral from being invoked on non settled or expired positions. these functions should include a check ensuring that settled == true(recommended) and that the position has not expired (i.e., expired > block.timestamp). This would prevent borrowers from exploiting these functions to extract collateral without repaying the debt and mitigate the risk of collateral theft from lenders.

## Proof of Concept

## Proof of Concept

run step below in settlement.t.sol.

* first thing create a settlement, a lend type settlement.
* after first step, rather than invoking settle, wrap the block.timestamp to (block.timestamp > expire).
* update oracle price to increase the collateral price(oracle is a mock currently)
* invoke the removeCollateral using the maker address(borrower).
