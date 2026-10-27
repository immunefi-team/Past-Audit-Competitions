# Attackathon \_ Fuel Network 32706 - \[Smart Contract - High] the function subtract in signed libs like

Submitted on Sun Jun 30 2024 13:19:53 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32706

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Block stuffing

## Description

## Brief/Intro

the function subtract used in signed libs to subtract two I8 number which are u8 numbers, this contract use bias mechanism to handle the signed values correctly since fuelVM did not support negative value, however the function subtract did not handle the case when `self.value < indent and other.value > indent` correctly which can block the subtracting logic when user sets the self as smaller number than the other.

## Vulnerability Details

checking the function below, we can see that there is situation when the self.value is smaller than indent and other.value is bigger or equal to the indent, if the other was bigger then an over/under flow can occur because it say self.value - (other - indent)\` which in this case the other is bigger than the self and the fuelVM will panic:

```sway
impl core::ops::Subtract for I8 {
    /// Subtract a I8 from a I8. Panics of overflow.
    fn subtract(self, other: Self) -> Self {
        let mut res = Self::new();
        
        if self.underlying >= Self::indent()
            && other.underlying >= Self::indent()
        {
            if self.underlying > other.underlying {
                res = Self::from_uint(self.underlying - other.underlying + Self::indent());
            } else {
                res = Self::from_uint(self.underlying - (other.underlying - Self::indent()));
            }

        } else if self.underlying >= Self::indent()
            && other.underlying < Self::indent()
        {
            res = Self::from_uint(self.underlying - Self::indent() + other.underlying);

        } else if self.underlying < Self::indent()
            && other.underlying >= Self::indent()
        { //@audit 

            res = Self::from_uint(self.underlying - (other.underlying - Self::indent())); // PANIC

        } else if self.underlying < Self::indent()
            && other.underlying < Self::indent()
        {

            if self.underlying < other.underlying {
                res = Self::from_uint(other.underlying - self.underlying + Self::indent());
            } else {
                res = Self::from_uint(self.underlying + other.underlying - Self::indent());
            }
        }
        res
    }
}

```

the function above should only accept value that wrapped by calling the `I8...256.from_uint(u8... u64) because using` from`function itself can break the whole contract math functionality, this mean when user use a valid u8 value as self.value which is smaller than the`other.value\` and other value is bigger than indent which is 128 the subtract functionality will be blocked and cause panic to feulVM.

NOTE :: we tried to use from function but this will break the whole math functionality because if you wrap a u8 value using the from function then there no such a case that value is smaller than indent that can be executed which is a more critical issue if the team planning to use `from` function from signed integer rather than `from_uint`

## Impact Details

incorrect handle of the case when self.value < indent and other > indent can block the subtract functionality.

## References

add if check inside the else if `self.underlying < Self::indent()&& other.underlying >= Self::indent()` check to make sure the subtract executed without causing panic because of overflow or underflow.

## Proof of concept

## Proof of Concept

run this test directly in the I8.sw lib.

```sway
#[test(should_revert)] // revert b/c of over/underflow
fn test_underflow_sub() {
    // Setup: self is negative (less than bias), other is positive (greater than or equal to bias)
    let self_value = I8::from_uint(100); 
    let other_value = I8::from_uint(250);

    // Perform subtraction
    let result = I8::subtract(self_value, other_value);
}


```
