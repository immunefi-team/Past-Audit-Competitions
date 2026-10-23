# #46893 \[SC-High] settlement functionality can be break forever and blocking settlement actions.

**Submitted on Jun 5th 2025 at 23:59:49 UTC by @zeroK for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #46893
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Protocol insolvency
  * Block stuffing

## Description

## Brief/Intro

the function `createSettlement` allow users to create settlement, either lend type or borrow type, if its lend type then the borrower(maker) should invoke settle function, if borrow type then lender(maker) should invoke settle function, in both cases, both borrower and lender should give approve to settlement contract to spent tokens on behalf of them, this is required to complete settlement and to meet the TSI wallet implementation requirement(1 key with fireblocks, another one with caller, no action invoked if one of them not sign specific transaction). however there is another function that can be used by malicious user to break the whole settlement mechanism and prevent the protocol to work as expected, which is the `addCollateralBeforeSettle`, this function implemented to allow the caller to increase the collateralAmt for any loan position without actually transfer collateral to the contract or the borrower address, this open possibilities to malicious user to invoke this function to increase collateralAmt to high value that exceed borrower collateral amount that hold prevent any call to settlement actions no matter if the settlement type is lend or borrow.

## Vulnerability Details

the function createSettlement implemented as below, it checks approval for both sides, borrower or lender or both, depending on the settle type:

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

required check for allowance done correctly since if its lend type, caller who is lender must approved settlement contract to spent collateral amount plus the debt, and same for borrower in this case, the problem arises when the `addCollateralBeforeSettle` invoked:

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

this function implemented in a way, that allow the borrower to increase the collateral before settle invoked, sometime borrower need to do so because the price of collateral might drop and lead to checkHealth revert when settle invoked, however, this function is invokable by anyone, anyone can call this function for any loan they want, this lead to malicious user to invoke the attack below:

* settlement between bob and eve created, bob approve 1000 eth to settlement contract to send it to eve, and eve approved to transfers 2M worth of USDC to bob when settle invoked.
* alice recognized this, and invoke call to `addCollateralBeforeSettle` function, setting the addedAmount == uint(256).max -1 or any higher value than bob hold, alice will approve the contract to spend this value but she can remove the invoke after any call to addCollateralBeforeSettle, this is only required when the settle type is borrow, alice actually does not lose any value to break the whole settlement contract.
* the transferfrom revert since the borrower does not hold that much of collateral amount even if increase the allowance, this will affect users and TSI protocol since it break the core function of the protocol, and it can lead to protocol insolvency since it prevent any fee or reward to be sent to the protocol or the fee collector address.

the addCollateralBeforeSettle implemented in a way that it work well only when it get called by the loan maker address, why is that? because:

* the maker is the lender or borrower, this is forced since settle function force this rule and you can't set address different from both.
* if the maker is borrower, then its meaningless for borrower to add collateral to a loan position without holding enough collateral, just to break the settlement action, this have no effects for both sides or even for the protocol.
* if the maker is lender, then the maker can add more collateral, this to get more collateral to its address when the borrower address hold more collateral than the amount provided for current loan(borrower hold 2k eth but want to use 1k eth for this loan), but since the collateral get locked in lender address, there is no direct theft of the collateral(i still review the incentive for the lender to do so and increase collateral amount.

## Impact Details

malicious user can break the core settlement functionality by invoking the addCollateralBeforeSettle function for unsettled loans.

## Recommend

I could recommend adding a check to allow only the maker address of the loan to invoke the addCollateralBeforeSettle function. However, there might currently be other consequences of invoking the addCollateralBeforeSettle function. I will update the fix to address any additional issues that may exist.

## Proof of Concept

## Proof of Concept

run the poc step in settlement.t.sol:

* first thing create settlement by using the `testOneLenderToMultipleBorrowersSettlement`(can be modified to create one loan only)
* after that invoke the testAddCollateralBeforeSettlement function to add collateral to the loan we create, add collateral to make the loan.debtData.collateralAmount to be more than the borrower collateral balance.
* the two step above can be all invoked in one test function for smoother test implementation.
