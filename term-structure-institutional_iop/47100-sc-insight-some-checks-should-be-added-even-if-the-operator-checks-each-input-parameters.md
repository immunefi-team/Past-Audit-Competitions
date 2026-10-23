# #47100 \[SC-Insight] some checks should be added even if the operator checks each input parameters

**Submitted on Jun 8th 2025 at 23:37:04 UTC by @zeroK for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47100
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**

## Description

## Brief/Intro

the createSettlement, allow a caller(the taker) to create a settlement, however there is some checks that its better to be added even if the operator checks each input before signing it as the team mentioned, these check can be for the feeAmount to be higher than zero, and the takerAmt is not zero, the maturity is more than the expiry, the maturity or expiry are not zero, the loanIDs and loans length are equal.

## Vulnerability Details

there is no Vulnerability, just best security practice in the function below:

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

## Impact Details

some checks are better to be added in createSettlement function.

## References

some checks as below can be added:

```solidity
function createSettlement(.....) {
 require(LoanInfo.debtData.feeAmt > 0, "zero fee");
require(LoanInfo.debtData.maturity > SettleInfo.expiry, "expiry higher than maturity");
require(SettleInfo.takerAmt > 0, "zero taker amount")
//.......


}
```

## Proof of Concept

## Proof of Concept

no POC can be implemented since this is insight report only
