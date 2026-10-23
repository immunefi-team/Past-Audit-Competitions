# Attackathon \_ Fuel Network 32696 - \[Smart Contract - High] incorrect setting of non\_negative value i

Submitted on Sat Jun 29 2024 20:22:34 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32696

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the function ceil in all of IFP libs used to return smallest number that is equal or greater than the `self.underlying`, this function have crucial rule as it will be used in many Defi protocols that will built on fuel, however there is incorrect set of `non_negative` value when this function called and the `else` case run, as there is possibility of setting the `non_negative` value to true when its false in specific case `if ceil == UFP64::from(1)` but this value will not be set in the self when its returned and used by the third party or the `round` function itself when calling the ceil function.

## Vulnerability Details

let's take a look at the `ceil` function in the IFP128:

```sway
    pub fn ceil(self) -> Self {
        let mut underlying = self.underlying;
        let mut non_negative = self.non_negative;

        if self.non_negative {
            underlying = self.underlying.ceil();
        } else {

            let ceil = self.underlying.ceil();

            if ceil != self.underlying {
                underlying = ceil + UFP64::from(1);
                if ceil == UFP64::from(1) {
                    non_negative = true; // in this situation non negative should be true
                }
            } else {
                underlying = ceil;
            }
        }
        Self {
            underlying: underlying,
            non_negative: self.non_negative,// but we set the self.non_negative which is not the updated value above (shown in POC)
        }
    }
}

```

as shown above when the ceil value we get is equal to `UFP64::from(1)` then the non negative should be true, but in the Self we set the non negative value to the input that we use as `self` which is false in our case, this can lead to critical issue for defi protocols when they use ceil function or round function which returns ceil in specific situation:

```sway

 pub fn round(self) -> Self {
        let mut underlying = self.underlying;

        if self.non_negative {
            underlying = self.underlying.round();
            
        } else {
            let floor = self.underlying.floor();
            let ceil = self.underlying.ceil();
            let diff_self_floor = self.underlying - floor;
            let diff_ceil_self = ceil - self.underlying;
            underlying = if diff_self_floor > diff_ceil_self {
                floor
            } else {
                ceil //@audit return ceil Self value which sets incorrectly
            };
        }
        Self {
            underlying: underlying,
            non_negative: self.non_negative,
        }
    }
}
```

we believe this can be high issue but it can be downgraded to medium as its logical issue if we saw it as issue in lib but critical issue if we think about how can cause damage to protocols directly. (we set this as medium and leave the judging to immunefi team for upgrading the impact level)

## Impact Details

incorrect set of non\_negative value in ceil can cause trouble when its called/used

## References

while there is possibility of non\_negative value to get updated then the Self.non\_negative should be set to the let `non_negative` variable

## Proof of concept

## Proof of Concept

run this test right below the ceil function in IFP128.sw by runing forc test

```sway
#[test]
fn test_ceil_function() {
      // Set up a value that, when ceiled, becomes 1
    let tiny_fractional_value = UFP64::from_uint(0) + UFP64::from(1); // Just below 1 in fixed-point
    let negative_tiny_fractional_value = IFP128::from(tiny_fractional_value).sign_reverse();

    // Ensure the value is negative
    let mut underlying = negative_tiny_fractional_value.underlying;
    let mut non_negative = negative_tiny_fractional_value.non_negative;

    assert(underlying == negative_tiny_fractional_value.underlying);
    assert(non_negative == negative_tiny_fractional_value.non_negative);

    assert(negative_tiny_fractional_value.non_negative() == false);

    // Calculate the ceiling value
    let ceil = negative_tiny_fractional_value.underlying.ceil();
    assert(ceil != underlying);

    // Apply the ceil function
    let res = negative_tiny_fractional_value.ceil();

    // Check that non_negative is set to true due to ceil being 1
    assert(ceil == UFP64::from_uint(1));
    assert(res.non_negative != true); //this should be set to true not false but the implementation not work correctly
}


```
