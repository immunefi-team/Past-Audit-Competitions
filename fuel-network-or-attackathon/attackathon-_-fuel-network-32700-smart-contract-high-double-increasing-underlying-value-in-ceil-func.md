# Attackathon \_ Fuel Network 32700 - \[Smart Contract - High] double increasing underlying value in cei

Submitted on Sat Jun 29 2024 20:44:41 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32700

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Theft of unclaimed yield
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the function ceil meant to be used to round up value or by another meaning to return smallest value that equal to underlying value or greater, however there is a critical issue in the ceil function which double increase the value of the underlying when the non\_negative is false, this is possible because the function add from(1) twice to the underlying value. more described in Vulnerability Details.

## Vulnerability Details

let's take a look how the ceil function work:

```sway

        pub fn ceil(self) -> Self {
        let mut underlying = self.underlying;
        let mut non_negative = self.non_negative;

        if self.non_negative {
            underlying = self.underlying.ceil();
        } else {

            let ceil = self.underlying.ceil(); //@audit this will increase the value of underlying after calling turnc function (while self.underlying is same as let underlying so its like increase the underlying value)

            if ceil != self.underlying {
                underlying = ceil + UFP64::from(1); // this again will increase the underlying value by 1 which its increased before(ceil(increaed value by one) + increase by from(1))
                if ceil == UFP64::from(1) {
                    non_negative = true;
                }
            } else {
                underlying = ceil;
            }
        }
        Self {
            underlying: underlying,
            non_negative: self.non_negative,
        }
    }
}

```

as shown above the ceil function called for the self.underlying which this function implemented in the UFP64, the ceil function calls turnc to remove the fractional part or(lower bit) and then increase it by 1 /2 pow 32 as shown below:

```sway

    pub fn ceil(self) -> Self {
        if self.fract().underlying != 0 {
            let res = self.trunc() + UFP64::from_uint(1);
            return res;
        }
        return self;
    }
}

```

as explained this function will add from\_uint(1) to the self value after turnc called, the issue arises here where the the ceil function called for self.underlying and increase it and then when `ceil != self.underlying` met, it add the ceil value which increase to underlying variable plus from(1) which this lead to double increase the value of underlying when returned by ceil function and used by round function. THIS can be critical issue because it can lead to theft of tokens when used by any defi app or by round function as it round up the value more than once.

## Impact Details

double increasing the underlying value can lead to critical issue to arise.

## References

recommended to not increase the value by one when the ceil it self is called which mean the `ceil != self.underlying` is not necessary at all

## Proof of concept

## Proof of Concept

run this POC in IFP128.sw lib by running forc test

```sway

#[test]
fn test_ceil_double_iuncrease_function() {

    let newVal = IFP128::from(UFP64::from_uint(128) + UFP64::from(3)).sign_reverse();

    let mut underlying = newVal.underlying;
    let mut non_negative = newVal.non_negative;

    assert(underlying == newVal.underlying);
    assert(non_negative ==  newVal.non_negative);

    assert(newVal.non_negative() == false); // insure the else case run 


    let ceil = newVal.underlying.ceil();
    assert(ceil != underlying); // ensure ceil != self.underlying run 

    // Ensure the underlying value before calling ceil on IFP128
    let expected_underlying_before = ceil;


    let res = newVal.ceil();

    let expected_underlying_after = expected_underlying_before + UFP64::from(1);
    assert(res.underlying == expected_underlying_after);

} 
```
