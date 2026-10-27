# Attackathon \_ Fuel Network 33168 - \[Smart Contract - High] Incorrect Sign Determination In Multiply

Submitted on Sat Jul 13 2024 08:16:13 GMT-0400 (Atlantic Standard Time) by @Minato7namikazi for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33168

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

During my audit of the IFP (signed fixed-point number) implementations in the sway-libs , this vulnerability were identified. This could lead to incorrect calculations in smart contracts relying on this implementation and cause massive losses.

## Vulnerability Details

Incorrect Sign Determination in Multiply & Division Operations in IFP128

The division operation incorrectly determines the sign of the result. the current implementation uses a logical condition that is always false, resulting in division operations always producing a positive result, regardless of the signs of the operands.

& the condition

`(self.non_negative && !self.non_negative) || (!self.non_negative && self.non_negative)`

in multiply & divide functions :

```

impl core::ops::Divide for IFP128 {
    /// Divide a IFP128 by a IFP128. Panics if divisor is zero.
    fn divide(self, divisor: Self) -> Self {
        let non_negative = if (self.non_negative
            && !self.non_negative)
            || (!self.non_negative
            && self.non_negative)
        {
            false
        } else {
            true
        };
        Self {
            underlying: self.underlying / divisor.underlying,
            non_negative: non_negative,
        }
    }
}

```

is always false because it's checking if a number is both positive and negative at the same time, which is impossible

## Impact Details

### permanent freezing of funds

because this vulnerability can lead to miscalculations in financial operations, and any mathematical computations relying on signed division like

1. Incorrect token balances and transfers
2. Potential exploitation in smart contracts dealing with debt, loans, or any negative value representations

## Proof of concept

### Add those PoCs tests in the end of the ifp128.sw

```
#[test]
fn PoC_divide_sign_issue() {

    let number1 = IFP128::from_uint(1); 
    let positive = IFP128::from_uint(4);  // 4
    let negative = positive.sign_reverse();  // -4
    let negative1 = number1.sign_reverse();  // -1


    // Dividing a positive by a negative should result in a negative

    let result = positive / negative;
    

    // "4 / -4 should be negative, but got positive"
    // This assertion will fail with the original implementation

    assert(result < negative1);

}


#[test]
fn PoC_multiply_sign_issue() {

    // Create positive and negative numbers

    let positive = IFP128::from_uint(5);  // 5
    let negative = IFP128::from_uint(3).sign_reverse();  // -3

    // Test case 1: Positive * Negative

    let result1 = positive * negative;
    
    // "5 * -3 should be negative, but got positive"
    // This assertion will fail with the original implementation

    assert(!result1.non_negative());

}

```
