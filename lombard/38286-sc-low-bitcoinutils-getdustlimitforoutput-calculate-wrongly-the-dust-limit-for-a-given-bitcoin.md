# 38286 \[SC-Low] bitcoinutils getdustlimitforoutput calculate wrongly the dust limit for a given bitco

## #38286 \[SC-Low] BitcoinUtils.getDustLimitForOutput calculate wrongly the dust limit for a given Bitcoin script public key

**Submitted on Dec 30th 2024 at 09:27:53 UTC by @perseverance for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38286
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/LBTC/LBTC.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

## Description

### Brief/Intro

During the redeem transaction, LBTC contract calculate if the amount after fee is above DustLimit. If so, then user can redeem successfully. Otherwise, the transaction will revert.

<https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/LBTC/LBTC.sol#L476-L482>

```solidity
function redeem(bytes calldata scriptPubkey, uint256 amount) external {
        LBTCStorage storage $ = _getLBTCStorage();

        // removed for simplicity 
        uint64 fee = $.burnCommission;
        (
            uint256 amountAfterFee,
            bool isAboveFee,
            uint256 dustLimit,
            bool isAboveDust
        ) = _calcFeeAndDustLimit(scriptPubkey, amount, fee);
        

        if (!isAboveDust) {
            revert AmountBelowDustLimit(dustLimit);
        }

        emit UnstakeRequest(fromAddress, scriptPubkey, amountAfterFee);
    }

```

The internal \_calcFeeAndDustLimit will call BitcoinUtils.getDustLimitForOutput to calculate the Dust Limit

<https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/libs/BitcoinUtils.sol#L65-L88>

```solidity
function getDustLimitForOutput(
        OutputType outType,
        bytes calldata scriptPubkey,
        uint256 dustFeeRate
    ) internal pure returns (uint256 dustLimit) {

Line 70:        uint256 spendCost = BASE_SPEND_COST;

        if (
            outType == OutputType.P2TR ||
            outType == OutputType.P2WPKH ||
            outType == OutputType.P2WSH
        ) {
            // witness v0 and v1 has a cheaper payment formula
Line 78:            spendCost += WITNESS_INPUT_SIZE;
        } else {
            spendCost += NON_WITNESS_INPUT_SIZE;
        }

Line 83:        spendCost += scriptPubkey.length;

        // Calculate dust limit
Line 86:        dustLimit = Math.ceilDiv(spendCost * dustFeeRate, 1000);
    }

```

This DustLimit is related to Dust that is defined in terms of dustRelayFee, which has units satoshis-per-kilobyte. If users pay more in fees than the value of the output to spend something, then Bitcoin consider it dust. [See Reference](https://github.com/bitcoin/bitcoin/blob/43740f4971f45cd5499470b6a085b3ecd8b96d28/src/policy/policy.cpp#L28-L41)

**So if the amount after fee is less than the DustLimit on BitcoinBitcoin, then the output amount can not be spent.**

The check in LBTC is important to prevent users to loose Bitcoin due to Dust Limit when redeem Bitcoin. It is important to calculate correctly this DustLimit to prevent users to loose Bitcoin.

### The vulnerability

#### Vulnerability Details

The bug here is function getDustLimitForOutput caculates wrong Dust Limit.

Currently the protocol supports three types of scriptPubkey: P2TR, P2WPKH, P2WSH.

To simplify, I will explain this for P2WPKH.

For P2WPKH, the function getDustLimitForOutput will return dustLimit is 291 for default dustFeeRate 3000

```solidity
Line 70: spendCost = BASE_SPEND_COST = 49 

Line 78: spendCost += WITNESS_INPUT_SIZE 
WITNESS_INPUT_SIZE = 26 

=> spendCost = 49 + 26 = 75 

Line 83: spendCost += scriptPubkey.length 

For P2WPKH scriptPubkey.length = 22 
=>   spendCost = 75 + 22  = 97 

with dustFeeRate = 3000 

Line 86:        dustLimit = Math.ceilDiv(spendCost * dustFeeRate, 1000);

dustLimit = 97 * 3 = 291 

```

But according to the comment in reference <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/libs/BitcoinUtils.sol#L64> in Bitcoin implementation:

<https://github.com/bitcoin/bitcoin/blob/43740f4971f45cd5499470b6a085b3ecd8b96d28/src/policy/policy.cpp#L37-L41>

```solidity
    // A typical spendable segwit P2WPKH txout is 31 bytes big, and will
    // need a CTxIn of at least 67 bytes to spend:
    // so dust is a spendable txout less than
    // 98*dustRelayFee/1000 (in satoshis).
    // 294 satoshis at the default rate of 3000 sat/kvB.
```

So the DustLimit should be 294 for the dustFeeRate of 3000

So with the amountAfterfee is 292, 293, 294 is considered valid to unstake according to contract LBTC redeem function, but according to Bitcoin network that is still Dust. So if users redeem such amount, the amount will not be able to spendd on Bitcoin.

I double checked the documentation for Bitcoin and I see that the comment in Bitcoin is correct and understandable.

The Output structure of a transaction would contains 3 fields as below:

<https://learnmeabitcoin.com/technical/transaction/#structure-outputs>

```
Field	
Amount	:	8 bytes	: 	The value of the output in satoshis.
ScriptPubKey: 1 byte 
ScriptPubKey : 22 bytes for P2WPKH 

=> Total size of output: 31 bytes for P2WPKH 
```

The [Input structure](https://learnmeabitcoin.com/technical/transaction/#structure-inputs) contains 5 fields as below

```
TXID: 32 bytes
VOUT: 4 bytes 
ScriptSig Size: 1 bytes for P2WPKH :  should be 00 for P2WPKH  (Ref: https://learnmeabitcoin.com/technical/transaction/#structure-inputs-scriptsig-size )
ScriptSig: 0 byte for P2WPKH 
Sequence:  4 bytes 

Total size: 41 
```

Witness size: 107 bytes but with 75% discount, it is considered as 26 bytes.

Reference: <https://github.com/bitcoin/bitcoin/blob/43740f4971f45cd5499470b6a085b3ecd8b96d28/src/policy/policy.cpp#L55-L57>

```
        // sum the sizes of the parts of a transaction input
        // with 75% segwit discount applied to the script size.

        if (txout.scriptPubKey.IsWitnessProgram(witnessversion, witnessprogram)) {
        // sum the sizes of the parts of a transaction input
        // with 75% segwit discount applied to the script size.
        nSize += (32 + 4 + 1 + (107 / WITNESS_SCALE_FACTOR) + 4);
    }
```

Total input + witness = 41 + 26 = 67 bytes => Total size for dustLimit calculation should be 67 + 31 = 98 bytes. This is correct according to Bitcoin documentation.

To summarize, the documentation of Bitcoin shows that DustLimit for P2WPKH should be 294 for defaultFeeRate of 3000. But the getDustLimitForOutput returns 291. So this results in some amount after fee that are 292, 293, 294 are considered valid for Lombard LBTC to redeem, but consider as Dust on Bitcoin. **This will result in the output redeemed amount will not be able to spend on Bitcoin.**

## Impacts

## About the severity assessment

Bug Severity: Low

Impact category:

Contract fails to deliver promised returns, but doesn't lose value

User lost of money

Freeze of user fund on Bitcoin

Note: Although for the reported bug, user redeemed amount will not be spend, means lost forever. But since the amount difference is small and the probability that user redeem such small amount after fee is not high, so I think it is more appropriate to set the reported but as Low severity.

### Proof of Concept

## Proof of concept

Steps to reproduce the bug:

Step 1: The test code to show this bug:

```typescript
     it('Redeem when amount is just above dust limit for P2WPKH', async () => {
                const p2wpkh = '0x00143dee6158aac9b40cd766b21a1eb8956e99b1ff03';
                const burnCommission = await lbtc.getBurnCommission();

                // Start with a very small amount
                let amount = burnCommission + 1n;
                let isAboveDust = false;

                // Incrementally increase the amount until we find the dust limit
                while (!isAboveDust) {
                    amount += 1n;
                    [, isAboveDust] = await lbtc.calcUnstakeRequestAmount(
                        p2wpkh,
                        amount
                    );
                }

                // Now 'amount' is just above the dust limit. Let's use an amount 1 less than this.
                const amountJustAboveDustLimit = amount;
                // console.log (amountJustAboveDustLimit);
                console.log('Amount just above dust limit:', amountJustAboveDustLimit);

                await lbtc.mintTo(signer1.address, amountJustAboveDustLimit);
                const expectedAmountAfterFee = amount - BigInt(burnCommission);
                
                console.log('burnCommission:', burnCommission);

                let amountJustBelowDustLimit = amountJustAboveDustLimit - 1n;
                console.log('amountJustBelowDustLimit:', amountJustBelowDustLimit);
                await expect(
                    lbtc
                        .connect(signer1)
                        .redeem(p2wpkh, amountJustBelowDustLimit)
                ).to.be.revertedWithCustomError(lbtc, 'AmountBelowDustLimit');

                console.log('expectedAmountAfterFee:', expectedAmountAfterFee);
                await expect(
                    lbtc
                        .connect(signer1)
                        .redeem(p2wpkh, amountJustAboveDustLimit)
                ).to.emit(lbtc, 'UnstakeRequest')
                .withArgs(signer1.address, p2wpkh, expectedAmountAfterFee);
            });
```

Just copy the test case into test suite "Positive cases" in evm-smart-contracts\test\LBTC.ts

```typescript
describe('Positive cases', function () 
```

To run the test

```
 yarn hardhat test
```

Test log:

```log
Amount just above dust limit: 1292n
burnCommission: 1000n
amountJustBelowDustLimit: 1291n
expectedAmountAfterFee: 292n
        ✔ Redeem when amount is just above dust limit for P2WPKH (497ms)

```

Explanation:

So the Amount just above dust limit: 1292n . After the fee, the expectedAmountAfterFee is 292. With this amount, the redeem is succesful. But on Bitcoin, this amount is less than Dust Limit that is 294 so it will not be able to spend.
