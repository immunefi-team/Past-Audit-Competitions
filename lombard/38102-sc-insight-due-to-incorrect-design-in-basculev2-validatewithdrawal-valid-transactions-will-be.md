# #38102 \[SC-Insight] Due to incorrect design in \`BasculeV2::validateWithdrawal\` valid transactions wi

**Submitted on Dec 24th 2024 at 11:41:51 UTC by @MrMorningstar for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38102
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bascule/BasculeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When users or white listed addresses want to mint `LBTC` by providing proof of stake action happened they can do via following functions:

* `mint` (<https://github.com/lombard-finance/evm-smart-contracts/blob/13bfc98f72c116c014633488de35aae774e5417b/contracts/LBTC/LBTC.sol#L368>)
* `batchMint`(<https://github.com/lombard-finance/evm-smart-contracts/blob/13bfc98f72c116c014633488de35aae774e5417b/contracts/LBTC/LBTC.sol#L394>)
* `mintWithFee`(<https://github.com/lombard-finance/evm-smart-contracts/blob/13bfc98f72c116c014633488de35aae774e5417b/contracts/LBTC/LBTC.sol#L415>)
* `batchMintWithFee`(<https://github.com/lombard-finance/evm-smart-contracts/blob/13bfc98f72c116c014633488de35aae774e5417b/contracts/LBTC/LBTC.sol#L431>)

After certain checks passed they all call `_validateAndMint` function at some point, which furthermore calls `_confirmDeposit` that looks like this:

```js
    function _confirmDeposit(
        LBTCStorage storage self,
        bytes32 depositID,
        uint256 amount
    ) internal {
        IBascule bascule = self.bascule;
        if (address(bascule) != address(0)) {
            bascule.validateWithdrawal(depositID, amount);
        }
    }
```

As we can see the `_confirmDeposit` functions calls the `validateWithdrawal`(<https://github.com/lombard-finance/evm-smart-contracts/blob/13bfc98f72c116c014633488de35aae774e5417b/contracts/bascule/BasculeV2.sol#L272>) which looks like this:

```js
    function validateWithdrawal(
        bytes32 depositID,
        uint256 withdrawalAmount
    ) public whenNotPaused onlyRole(WITHDRAWAL_VALIDATOR_ROLE) {
        DepositState status = depositHistory[depositID];
        // Deposit found and not withdrawn
        if (status == DepositState.REPORTED) {
            depositHistory[depositID] = DepositState.WITHDRAWN;
            emit WithdrawalValidated(depositID, withdrawalAmount);
            return;
        }
        // Already withdrawn
        if (status == DepositState.WITHDRAWN) {
            revert AlreadyWithdrawn(depositID, withdrawalAmount);
        }
        // Not reported
        if (withdrawalAmount >= validateThreshold()) {
            // We disallow a withdrawal if it's not in the depositHistory and
            // the value is above the threshold.
            revert WithdrawalFailedValidation(depositID, withdrawalAmount);
        }
        // We don't have the depositID in the depositHistory, and the value of the
        // withdrawal is below the threshold, so we allow the withdrawal without
        // additional on-chain validation.
        //
        // Unlike in original Bascule, this contract records withdrawals
        // even when the validation threshold is raised.
        depositHistory[depositID] = DepositState.WITHDRAWN;
        emit WithdrawalNotValidated(depositID, withdrawalAmount);
    }
```

## Vulnerability Details

The issue arise with this part of the code:

```js
        // Not reported
        if (withdrawalAmount >= validateThreshold()) {
            // We disallow a withdrawal if it's not in the depositHistory and
            // the value is above the threshold.
            revert WithdrawalFailedValidation(depositID, withdrawalAmount);
        }
```

As we can see clearly from the commend and intended design the purpose is to not allow withdrawal if report is not found or created by deposit reporter. The problem is that the intended design is to not allow withdrawals that are above threshold but due to current design it will revert even if the withdrawal amount is equal to the threshold.

## Impact Details

Valid transactions will be reverted, which will make protocol unable to mint tokens in valid scenarios

## Recommendation

Make the following change in `validateWithdrawal`:

```diff
    function validateWithdrawal(
        bytes32 depositID,
        uint256 withdrawalAmount
    ) public whenNotPaused onlyRole(WITHDRAWAL_VALIDATOR_ROLE) {
        DepositState status = depositHistory[depositID];
        // Deposit found and not withdrawn
        if (status == DepositState.REPORTED) {
            depositHistory[depositID] = DepositState.WITHDRAWN;
            emit WithdrawalValidated(depositID, withdrawalAmount);
            return;
        }
        // Already withdrawn
        if (status == DepositState.WITHDRAWN) {
            revert AlreadyWithdrawn(depositID, withdrawalAmount);
        }
        // Not reported
-        if (withdrawalAmount >= validateThreshold()) {
+        if (withdrawalAmount > validateThreshold()) {
            // We disallow a withdrawal if it's not in the depositHistory and
            // the value is above the threshold.
            revert WithdrawalFailedValidation(depositID, withdrawalAmount);
        }
        // We don't have the depositID in the depositHistory, and the value of the
        // withdrawal is below the threshold, so we allow the withdrawal without
        // additional on-chain validation.
        //
        // Unlike in original Bascule, this contract records withdrawals
        // even when the validation threshold is raised.
        depositHistory[depositID] = DepositState.WITHDRAWN;
        emit WithdrawalNotValidated(depositID, withdrawalAmount);
    }
```

## Proof of Concept

## Proof of Concept

Paste the following test in `Bascule.ts`(<https://github.com/lombard-finance/evm-smart-contracts/blob/main/test/Bascule.ts>):

```js
        it('Does not allow withdrawals equal to threshold', async () => {
            // grant guardian role
            // eslint-disable-next-line new-cap
            const guardianRole = await bascule.VALIDATION_GUARDIAN_ROLE();
            await expect(
                bascule.connect(admin).grantRole(guardianRole, guardian)
            )
                .to.emit(bascule, 'RoleGranted')
                .withArgs(guardianRole, guardian, admin);

            // raise validation threshold
            await expect(bascule.connect(guardian).updateValidateThreshold(33))
                .to.emit(bascule, 'UpdateValidateThreshold')
                .withArgs(0, 33);

            // add some deposits
            const one = new Uint8Array(32).fill(1);
            const two = new Uint8Array(32).fill(2);
            const three = new Uint8Array(32).fill(3);
            const reportId = ethers.randomBytes(32);
            await expect(
                bascule
                    .connect(depositReporter)
                    .reportDeposits(reportId, [one, two])
            )
                .to.emit(bascule, 'DepositsReported')
                .withArgs(reportId, 2);
                // withdrawal amount = threshold
                await expect(
                    bascule
                        .connect(withdrawalValidator)
                        .validateWithdrawal(three, 33)
                )
                    .to.be.revertedWithCustomError(
                        bascule,
                        'WithdrawalFailedValidation'
                    )
                    .withArgs(three, 33);
                });
```

And execute the following command in terminal:

```
yarn hardhat test --grep "Does not allow withdrawals equal to threshold"
```

The test will pass which prove that the intended design (to not allow to withdraw amounts that are bigger than threshold) is not correctly implemented, which will revert valid and desired deposits by the protocol and unable to mint in those scenarios.
