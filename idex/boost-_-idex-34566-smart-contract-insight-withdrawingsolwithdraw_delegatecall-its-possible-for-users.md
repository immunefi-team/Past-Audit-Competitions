# Boost \_ IDEX 34566 - \[Smart Contract - Insight] Withdrawingsolwithdraw\_delegatecall - Its possible f

Submitted on Thu Aug 15 2024 22:52:26 GMT-0400 (Atlantic Standard Time) by @OxSCSamurai for [Boost | IDEX](https://immunefi.com/bounty/boost-idex/)

Report ID: #34566

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/idexio/idex-contracts-ikon/blob/main/contracts/libraries/Withdrawing.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

## Summary:

* The problem is that the current logic allows for users to withdraw an amount equal to the `maximumGasFee`, which will cause the user to receive nothing but still pay withdrawal transaction fees.
* There seems to be no checks to prevent this from happening.

#### The affected function:

<https://github.com/idexio/idex-contracts-ikon/blob/4b4e154cb28269972c87f6c8864bf65cbea8220e/contracts/libraries/Withdrawing.sol#L98-L154>

The affected require statement:

```solidity
    require(
      arguments.withdrawal.gasFee <= arguments.withdrawal.maximumGasFee &&
        arguments.withdrawal.maximumGasFee <= arguments.withdrawal.grossQuantity, /// @audit-issue allows for users to receive zero withdrawal amounts while still paying fees...
      "Excessive withdrawal fee"
    );
```

## Impact Details

## Impact:

#### Impact in scope:

* `Contract fails to deliver promised returns, but doesn't lose value` - low severity >>> Absolutely no user ever expects to receive zero amount when completing a withdrawal transaction successfully.
* Any user would expect to receive a withdrawn amount of > 0, therefore the contract fails to deliver on promised or expected returns or behaviour in terms of withdrawal functionality.

#### Impact Analysis:

Expected Returns:

* Users generally expect that when they initiate a withdrawal, they will receive a certain value or asset in return for their transaction. In this case, if a user attempts to withdraw assets and receives zero value, the contract is failing to deliver the expected returns.
* Additionally, the require statement's error/revert message states `Excessive withdrawal fee`, which is when withdrawal fee is greater than the withdraw amount. This means the user will NOT get > 0 out in withdrawal amounts, so that's why the require check is there to *prevent* zero amount withdrawals. But the require check unfortunately fails to prevent zero amount withdrawals for case `arguments.withdrawal.maximumGasFee == arguments.withdrawal.grossQuantity`.
* So there's an inconsistency here: on one hand the require check is there to prevent a situation where a user withdraws and receives a zero amount of tokens, but the check fails to account for all possible cases where withdrawn amount will end up being zero.

User Experience:

* The ability to withdraw zero value while still being charged fees can negatively impact the user experience. Users may feel that they are being penalized for attempting to withdraw their assets, even if the contract itself is not losing value.
* Such scenarios can affect the trust and credibility of the protocol. Users may question the fairness of the fee structure and the overall design of the contract if they find themselves in situations where they incur costs without receiving any tangible benefits.

## References

<https://github.com/idexio/idex-contracts-ikon/blob/4b4e154cb28269972c87f6c8864bf65cbea8220e/contracts/libraries/Withdrawing.sol#L98-L154>

## Proof of concept

## Proof of Concept

## PoC Tests:

#### The existing hardhat/typescript test I used with slight modifications:

* `idex-contracts-ikon/test/withdrawals.ts`:

```typescript
    it('should revert on excessive withdrawal fee', async function () {
      await expect(
        exchange
          .connect(dispatcherWallet)
          .withdraw(
            ...getWithdrawArguments(withdrawal, '0.70000000', signature),
          ),
      ).to.eventually.be.rejectedWith(/excessive withdrawal fee/i);

      //withdrawal.maximumGasFee = '1.10000000';
      withdrawal.maximumGasFee = '1.00000000'; /// @audit added for PoC/testing purposes >>> withdrawal amount is 1. Due to bug, it should not revert here but allow user to withdraw ZERO amount.
      signature = await traderWallet.signTypedData(
        ...getWithdrawalSignatureTypedData(
          withdrawal,
          await exchange.getAddress(),
        ),
      );
      await expect(
        exchange
          .connect(dispatcherWallet)
          .withdraw(
            ...getWithdrawArguments(withdrawal, '0.10000000', signature),
          ),
      ).to.eventually.be.rejectedWith(/excessive withdrawal fee/i);
    });
```

#### Test 1: Demonstrating that a user can receive zero amount after trying to withdraw > 0 amount, and still be charged fees:

Using:

```solidity
    require(
      arguments.withdrawal.gasFee <= arguments.withdrawal.maximumGasFee &&
        arguments.withdrawal.maximumGasFee <= arguments.withdrawal.grossQuantity,
        //arguments.withdrawal.maximumGasFee < arguments.withdrawal.grossQuantity, /// @audit added for PoC/testing purposes
      "Excessive withdrawal fee"
    );
```

#### Test 1 results:

```hardhat
$ npm run test test/withdrawals.ts -- --grep "should revert on excessive withdrawal fee"

> @idexio/idex-contracts-ikon@1.0.0 test
> COVERAGE=1 hardhat test test/withdrawals.ts --grep should revert on excessive withdrawal fee


Compiled 3 Solidity files successfully (evm target: paris).


  Exchange
    withdraw
      1) should revert on excessive withdrawal fee


  0 passing (3s)
  1 failing

  1) Exchange
       withdraw
         should revert on excessive withdrawal fee:
     AssertionError: expected promise to be rejected with an error matching /excessive withdrawal fee/i but it was fulfilled with ContractTransactionResponse{ …(18) }
```

* The withdraw transaction was successful, and user has received zero withdrawal amount but was still charged fees...

#### Test 2: Demonstrating the bugfix, now the user shouldnt be able to receive zero withdrawal amount. The transaction should revert.

* Yes, the user probably still pays miner/validator fees here(for reverted/failed txs) but this is expected.
* But it's *not* expected to have a successful withdrawal transaction and receive zero withdrawal amount. Using:

```solidity
    require(
      arguments.withdrawal.gasFee <= arguments.withdrawal.maximumGasFee &&
        //arguments.withdrawal.maximumGasFee <= arguments.withdrawal.grossQuantity,
        arguments.withdrawal.maximumGasFee < arguments.withdrawal.grossQuantity, /// @audit added for PoC/testing purposes
      "Excessive withdrawal fee"
    );
```

#### Test 2 results:

```hardhat
$ npm run test test/withdrawals.ts -- --grep "should revert on excessive withdrawal fee"

> @idexio/idex-contracts-ikon@1.0.0 test
> COVERAGE=1 hardhat test test/withdrawals.ts --grep should revert on excessive withdrawal fee


Compiled 3 Solidity files successfully (evm target: paris).


  Exchange
    withdraw
      ✔ should revert on excessive withdrawal fee


  1 passing (3s)
```

* The withdraw tx failed/reverted, as it should.
* With the bugfix implemented the withdraw tx now reverts, preventing user from receiving zero withdraw amount.

## Suggested bugfix:

* To prevent users from withdrawing an amount that results in zero value after fees, ensure that the gross quantity is *greater* than the maximum gas fee while also verifying that the specified gas fee does not exceed the maximum allowable gas fee:

```solidity
    require(
      arguments.withdrawal.gasFee <= arguments.withdrawal.maximumGasFee &&
-       arguments.withdrawal.maximumGasFee <= arguments.withdrawal.grossQuantity,
+       arguments.withdrawal.maximumGasFee < arguments.withdrawal.grossQuantity,
      "Excessive withdrawal fee"
    );
```
