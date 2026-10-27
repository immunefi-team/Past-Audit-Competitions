# #46819 \[SC-Critical] direct theft of users funds when expired loan get liquidated

**Submitted on Jun 4th 2025 at 21:25:24 UTC by @zeroK for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #46819
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Currently, the `Settlement.sol` contract allows either the lender or the borrower to create a settlement by invoking the `createSettlement` function. This function is designed to work as intended only if the operator properly validates all input parameters.

When creating a settlement, if the caller is the lender (indicated by takerType == lender), then only the borrower or maker can execute the settlement. This design ensures that both parties remain aligned and that the agreement is mutually acknowledged. However, the borrower can choose to not agree to the settlement once it’s created and simply let it expire. There is a significant incentive for the borrower to do this. By refusing the settlement, the borrower can effectively exploit the system to seize the lender’s collateral. This collateral might originally come from another honest borrower, plus any liquidation incentives, and the borrower can even reclaim their own collateral if they fully repay the debt. At first glance, this might appear fair since the lender still receives the repaid debt amount. However, as explained in the Vulnerability Details section, this setup can be manipulated, leading to unfair outcomes and potential losses for the lender.

## Vulnerability Details

first thing, imagine a settlement agreed on by operator between alice(malicious borrower) and bob(victim lender) created by invoking the createSettlement function, the lender or taker which is bob created the settlement:

```solidity
    function createSettlement(
        string memory _settlementId,
        SettleInfo calldata _settleInfo, 
        string[] memory _loanIds,
        LoanInfo[] calldata _loans,  
        bytes calldata _signature 
    ) external nonReentrant {
        //PART 1: Verify signature
        bytes32 digest = _getSettlementHash(_settlementId, _settleInfo, _loanIds, _loans);
        if (!SignatureChecker.isValidSignatureNow(_operator, digest, _signature)) {
            revert InvalidSignature();
        } 
        //PART 2: Create sanity checks
        bytes32 settlementId = _settlementId.toBytes32();
        if (settlements[settlementId].taker != address(0)) {
            revert SettlementAlreadyExists(settlementId);
        } // if the settlement already exists, revert
        if (_settleInfo.expiryTime < block.timestamp) {
            revert SettlementExpired(block.timestamp, _settleInfo.expiryTime);
        } // if the settlement is in past(it can be equal)
        if (_settleInfo.taker != msg.sender) {
            revert TakerNotMatched(settlementId, _settleInfo.taker, msg.sender);
        } // taker must be the caller
        if (_loans.length == 0) {
            revert EmptyLoan();
        }
        // PART 3: Create settlement and loans
        uint256 totalCollateralAmt;
        uint256 totalDebtAmt;
        address debtToken = _loans[0].debtTokenAddr; // debt token addr for first loan(USDC)
        address collateralToken = _loans[0].collateralTokenAddr; // collateral token addr for first loan(ETH)
        
        for (uint256 i = 0; i < _loanIds.length; i++) {
            bytes32 loanId = _loanIds[i].toBytes32(); 
            LoanInfo memory loan = _loans[i];
            if (loans[loanId].maker != address(0)) {
                revert LoanAlreadyExisted(loanId);
            } // if the loan created before, revert

            loan.settlementId = settlementId; //set the settlement id for each loan
            loan.settled = false;
            loans[loanId] = loan; //UPDATE storage
            totalCollateralAmt += loan.debtData.collateralAmt;
            totalDebtAmt += loan.debtData.debtAmt;

            if (_settleInfo.takerType == TakerType.BORROW && loan.collateralTokenAddr != collateralToken) {
                revert TakerNotMatched(loanId, collateralToken, loan.collateralTokenAddr);

            } //if the taker is borrowing then check if the all loan have same collateral token 
            else if (_settleInfo.takerType == TakerType.LEND && loan.debtTokenAddr != debtToken) {
                revert TakerNotMatched(loanId, debtToken, loan.debtTokenAddr);
            } //if taker is lending then check if the debt token is same for all loans
        }
        if (_settleInfo.takerType == TakerType.LEND) {

            uint256 allowance = IERC20(collateralToken).allowance(msg.sender, address(this));
            if (allowance < Constants.ALLOWANCE_SCALE * totalCollateralAmt) {
                revert TokenAllowanceInsufficient(allowance, Constants.ALLOWANCE_SCALE * totalCollateralAmt);
            } // lender must approve this contract before doing anything

            allowance = IERC20(debtToken).allowance(msg.sender, address(this));
            if (allowance < totalDebtAmt) {
                revert TokenAllowanceInsufficient(allowance, totalDebtAmt);
            } // lender must approve this contract before doing anything(for debt token)
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

We can see that Bob (the msg.sender, acting as the taker/lender) grants full approval to the Settlement contract to manage the total collateral amount he has received from a later settlement call by the borrower. For example, let’s say the collateral amount is 5,000 ETH when 1 ETH equals 2,000 USDC. Bob now waits for Alice (the borrower) to invoke the settle function to complete the settlement, update the settlement status to Settled, and transfer the collateral while receiving the debt amount. However, Alice doesn’t call the settle function; instead, she waits for the settlement to expire (which happens in 1 day in our example). Rather than settling, she waits until the loan matures (10 days in this example).

It’s important to note that no money or collateral is actually transferred between Alice and Bob until the settle function is invoked. This function is the only way to finalize the settlement, as shown in the implementation below

```solidity

        function settle(string memory _loanId) public nonReentrant {
        bytes32 loanId = _loanId.toBytes32();
        LoanInfo memory loanInfo = loans[loanId]; // get the loan info data

        if (loanInfo.maker == address(0)) {
            revert LoanNotFound(loanId);
        }
        if (loanInfo.maker != msg.sender) {
            revert MakerNotMatched(loanInfo.settlementId, loanId, loanInfo.maker, msg.sender);
        } //only maker can invoke settle
        if (loanInfo.settled) {
            revert LoanAlreadySettled(loanId);
        }

        SettleInfo memory settleInfo = settlements[loanInfo.settlementId];

        if (settleInfo.expiryTime < block.timestamp) {
            revert SettlementExpired(block.timestamp, settleInfo.expiryTime);
        }

        if (settleInfo.takerType == TakerType.BORROW) {
            if (loanInfo.lender != loanInfo.maker) {
                revert LenderNotMatched(loanId, loanInfo.lender, loanInfo.maker);
            } //lender == msg.sender
            if (loanInfo.borrower != settleInfo.taker) {
                revert BorrowerNotMatched(loanId, loanInfo.borrower, settleInfo.taker);
            } // borrower == taker
            uint256 allowance = IERC20(loanInfo.collateralTokenAddr).allowance(msg.sender, address(this)); // msg.sender is maker
            if (allowance < Constants.ALLOWANCE_SCALE * loanInfo.debtData.collateralAmt) {
                revert TokenAllowanceInsufficient(
                    allowance, Constants.ALLOWANCE_SCALE * loanInfo.debtData.collateralAmt
                );
            }
        } else {
            if (loanInfo.borrower != loanInfo.maker) {
                revert BorrowerNotMatched(loanId, loanInfo.borrower, loanInfo.maker);
            } // borrower == msg.sender
            if (loanInfo.lender != settleInfo.taker) {
                revert LenderNotMatched(loanId, loanInfo.lender, settleInfo.taker);
            } // lender == taker
        }
        loanInfo.checkHealth(_oracle);

        loans[loanId].settled = true;

        IERC20(loanInfo.debtTokenAddr).safeTransferFrom(
            loanInfo.lender, loanInfo.borrower, loanInfo.debtData.borrowedAmt
        );
        IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(
            loanInfo.borrower, loanInfo.lender, loanInfo.debtData.collateralAmt
        );
        IERC20(loanInfo.debtTokenAddr).safeTransferFrom(loanInfo.lender, _feeCollector, loanInfo.debtData.feeAmt);

        emit Settled(msg.sender, loanId);
    }

```

now after 10 days, maturity passed, and before this, bob got another settlement for the same amount with eve as borrower, bob now hold 5000 eth, and eve got 9M USDC, now alice invoke liquidity function with the loanID equal to the expired one:

```solidity
    function liquidate(string memory _loanId, uint256 liquidationAmt) external nonReentrant {
        bytes32 loanId = _loanId.toBytes32();
        LoanInfo memory loanInfo = loans[loanId];
        (uint256 collateralToLiquidator, uint256 collateralToProtocol) =
            loanInfo.liquidate(loanId, _oracle, _minimumDebtValue, liquidationAmt);

        IERC20(loanInfo.debtTokenAddr).safeTransferFrom(msg.sender, loanInfo.lender, liquidationAmt);
        IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(loanInfo.lender, msg.sender, collateralToLiquidator);
        IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(loanInfo.lender, _feeCollector, collateralToProtocol);

        if (loanInfo.debtData.debtAmt == 0) {
            IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(
                loanInfo.lender, loanInfo.borrower, loanInfo.debtData.collateralAmt
            );
            delete loans[loanId];
        } else {
            loans[loanId] = loanInfo;
        }
        emit Liquidated(
            loanId,
            msg.sender,
            liquidationAmt,
            collateralToLiquidator,
            collateralToProtocol,
            loanInfo.debtData.collateralAmt
        );
    }

    function liquidate(
        LoanInfo memory loanInfo,
        bytes32 loanId,
        IOracle oracle,
        uint256 minimumDebtValue,
        uint256 amount
    ) internal view returns (uint256 collateralToLiquidator, uint256 collateralToProtocol) {
        (bool liquidateable,, uint256 maxLiquidationAmt, PriceInfo memory collateralPrice, PriceInfo memory debtPrice) =
            liquidaitonInfo(loanInfo, oracle, minimumDebtValue);
        if (!liquidateable) {
            revert SettlementErrors.CannotLiquidate(loanId);
        }
        if (
            amount > loanInfo.debtData.debtAmt
                || amount > maxLiquidationAmt * (Constants.DECIMAL_BASE + Constants.MAX_LTV_BUFFER) / Constants.DECIMAL_BASE
        ) {
            revert SettlementErrors.LiquidationAmtExceedsMax(amount, maxLiquidationAmt);
        }
        uint256 ltvBefore = LoanLib.calculateLtv(loanInfo, collateralPrice, debtPrice);
        (uint256 pdToPc, uint256 denominator) = _calculatePaToPbUpround(debtPrice, collateralPrice);
        uint256 removedCollateralAmt =
            amount * pdToPc * collateralPrice.tokenDenominator / (denominator * debtPrice.tokenDenominator);

        if (removedCollateralAmt > loanInfo.debtData.collateralAmt) {
            revert SettlementErrors.CollateralRemovedAmtExceedsCollateralAmt(
                loanInfo.debtData.collateralAmt, removedCollateralAmt
            );
        }
        collateralToLiquidator =
            removedCollateralAmt + (removedCollateralAmt * Constants.REWARD_TO_LIQUIDATOR) / Constants.DECIMAL_BASE;
        collateralToProtocol = (removedCollateralAmt * Constants.REWARD_TO_PROTOCOL) / Constants.DECIMAL_BASE;

        if (collateralToLiquidator >= loanInfo.debtData.collateralAmt) {
            collateralToLiquidator = loanInfo.debtData.collateralAmt;
            collateralToProtocol = 0;

        } else if (collateralToLiquidator + collateralToProtocol >= loanInfo.debtData.collateralAmt) {
            collateralToProtocol = loanInfo.debtData.collateralAmt - collateralToLiquidator;
        }
        loanInfo.debtData.collateralAmt =
            loanInfo.debtData.collateralAmt - collateralToLiquidator - collateralToProtocol;
        loanInfo.debtData.debtAmt -= amount;

        uint256 ltvAfter = calculateLtv(loanInfo, collateralPrice, debtPrice);
        if (ltvAfter == 0) {
            return (collateralToLiquidator, collateralToProtocol);
        }
        if (ltvAfter >= ltvBefore || ltvAfter < loanInfo.debtData.mltv - Constants.MAX_LTV_BUFFER) {
            revert SettlementErrors.LtvInvalidAfterLiquidation(ltvAfter, ltvBefore);
        }
    }


    function liquidaitonInfo(LoanInfo memory loanInfo, IOracle oracle, uint256 minimumDebtValue)
        internal
        view
        returns (
            bool liquidateable, // is liquidateable
            bool deliverable, // is deliverable
            uint256 maxLiquidationAmt,
            PriceInfo memory collateralPrice,
            PriceInfo memory debtPrice
        )
    {
        (collateralPrice, debtPrice) = getPriceInfos(loanInfo, oracle);
        uint256 ltv = LoanLib.calculateLtv(loanInfo, collateralPrice, debtPrice);
        if (loanInfo.debtData.maturity <= block.timestamp) { // if maturity reached
            if (ltv >= Constants.DECIMAL_BASE) {
                deliverable = true;
            } else {
                liquidateable = true;
                maxLiquidationAmt = loanInfo.debtData.debtAmt; // for mature, set all debt amount
            } //even if collateral is much more than debt, it is liquidateable because its mature
        } else {
            if (ltv >= Constants.DECIMAL_BASE) {
                deliverable = true;
            } else if (ltv >= loanInfo.debtData.lltv) {
                liquidateable = true;
                maxLiquidationAmt = _calculateMaxLiquidationAmt(loanInfo, collateralPrice, debtPrice);
                uint256 remainningDebtAmt = loanInfo.debtData.debtAmt - maxLiquidationAmt;
                if (remainningDebtAmt > 0 && remainningDebtAmt < _calculateMinimumDebtAmt(minimumDebtValue, debtPrice))
                {
                    maxLiquidationAmt = loanInfo.debtData.debtAmt;
                }
            }
        }
    }




```

so what is the benefits for alice to do this since she's forced to pay all debt to lender? alice will get more eth or collateral than the debt she pays plus she affect bob new loan position and prevent the loan position for eve to be completed normally and lead to liquidation which might revert, so what alice get is:

* first she invoke the liquidate with: loanID = loan id between him and bob, and liquidationAmt = full repay the debt amount which is 9.5M USDC.
* the internal liquidate function invoked, it first check if the position is matured in line below:

```solidity
        if (loanInfo.debtData.maturity <= block.timestamp) { // if maturity reached
else {
                liquidateable = true;
                maxLiquidationAmt = loanInfo.debtData.debtAmt; // for mature, set all debt amount
            } //even if collateral is much more than debt, it is liquidateable because its mature
```

* the max liquidation is the full amount of the debt or 9.5M USDC, then the internal liquidate function calculate the eth that the liquidator should get in return(alice or the borrower in our case) in line below:

```solidity

        uint256 removedCollateralAmt =
            amount * pdToPc * collateralPrice.tokenDenominator / (denominator * debtPrice.tokenDenominator);


```

* she gets around 4750 in total of 5k eth in collateral in bob address, and plus she gets the liquidation incentive in line below:

```solidity
        collateralToLiquidator =
            removedCollateralAmt + (removedCollateralAmt * Constants.REWARD_TO_LIQUIDATOR) / Constants.DECIMAL_BASE;
        collateralToProtocol = (removedCollateralAmt * Constants.REWARD_TO_PROTOCOL) / Constants.DECIMAL_BASE;


```

* the incentive can be around another 30 eth and 40 eth for protocol as fee(nearly value), and debtAmt become 0(since we payed fully) and then after that the liquidate function will invoke transfer actions as below:

```solidity
        IERC20(loanInfo.debtTokenAddr).safeTransferFrom(msg.sender, loanInfo.lender, liquidationAmt);
        IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(loanInfo.lender, msg.sender, collateralToLiquidator);
        IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(loanInfo.lender, _feeCollector, collateralToProtocol);

        if (loanInfo.debtData.debtAmt == 0) {
            IERC20(loanInfo.collateralTokenAddr).safeTransferFrom(
                loanInfo.lender, loanInfo.borrower, loanInfo.debtData.collateralAmt
            );
            delete loans[loanId];


```

* alice transfer 9.5 million to bob(flashloan can be used without taking fees), alice who is msg.sender gets around 4800 eth(9.6M in USDC) and the protocol gets the fee(50 eth) and any remaining collateral will be sent to borrower because debt is 0, which is alice herself. in total alice got around +1M USDC profit.
* not only this, this action affect the agreement(the loan position) between bob and eve, because when eve want to payback the debt to get her collateral, the repay function will revert, because bob does not hold the collateral to pay eve back when repay function invoked, even same can be true for liquidation, in this case eve can not get back her collateral until bob get collateral to his wallet, liquidation process revoked by bots and reverts, it might get delievered which lead to same result, reverting, in this case the whole protocol can be affected by this root cause.

this took from this example in TSI docs with full liquidation : <https://docs.institutional.ts.finance/features/liquidation-and-physical-delivery#liquidation-penalty-calculation>

## Impact Details

expired loan never removed or deleted which can be used to take theft of lender funds

## Recommend

simple fix is as below:

* if the loan expired, instead of reverting when settle called, delete the loan, even if the caller is not the maker.
* allow bot only to invoke the liquidate function, one address can be created and act like a bot by checking events emitting.

## Proof of Concept

## Proof of Concept

the POC is simple, in the settlement.t.sol do the follow:

* create a loan similar to how the testCreateSettlement do and set expire 1 day and maturity 11 days.
* wrap the timestamp(vm.wrap(block.timestamp + 12 days).
* and then invoke the liquidate for the expired loan. all this should be done in one test function
