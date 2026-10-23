# Attackathon \_ Fuel Network 32412 - \[Smart Contract - Insight] the IFP divide functions does not have

Submitted on Thu Jun 20 2024 14:54:21 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32412

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Unbounded gas consumption
* Block stuffing
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the function `divide` in the ifp libs are used to divide two number, however this function mentioned that the function will panic when divisor is zero rather than revert with helpful error messgae, this can cause trouble for users interfaces in case of debugging. an `assert` is implemented in UFP libs which revert in this case rather than panic the whole fuelVM.

## Vulnerability Details

the `divide` in IFP libs are implemented as below:

```sway
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

as shown above the call will panic if the divisor is zero and no check implemented to revert with reason rather than panic similar to ufp libs:

```solidity

impl core::ops::Divide for UFP64 {
    /// Divide a UFP64 by a UFP64. Panics if divisor is zero.
    fn divide(self, divisor: Self) -> Self {
        let zero = UFP64::zero();
        assert(divisor != zero);

        let denominator = U128::from((0, Self::denominator()));
        // Conversion to U128 done to ensure no overflow happen
        // and maximal precision is avaliable
        // as it makes possible to multiply by the denominator in 
        // all cases
        let self_u128 = U128::from((0, self.underlying));
        let divisor_u128 = U128::from((0, divisor.underlying));

        // Multiply by denominator to ensure accuracy 
        let res_u128 = self_u128 * denominator / divisor_u128;

        if res_u128.upper() != 0 {
            // panic on overflow
            revert(0);
        }
        Self {
            underlying: res_u128.lower(),
        }
    }
}

```

## Impact Details

the divide function panics when the divisor is zero rather than revert with helpful error message.

## References

its recommended to avoid panics in fuelVM by using assert, implement the ifp libs similar to ufp libs.

## Proof of concept

## Proof of Concept

```sway
contract;

use sway_libs::fixed_point::{ifp128::IFP128, ifp256::IFP256, ifp64::IFP64,};

abi MyContract {
    fn test_function();

     

}

impl MyContract for Contract {

    fn test_function() {
       let num  = IFP128::from_uint(10u64); 
       let num2 = IFP128::from_uint(0); 

       IFP128::divide(num, num2);             
    }
   
}

 #[test(should_revert)]
    fn test_function_contract() {
        
        let caller = abi(MyContract, CONTRACT_ID);
        let result = caller.test_function {}();
    }


```
