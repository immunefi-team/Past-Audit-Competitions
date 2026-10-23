# Attackathon \_ Fuel Network 32276 - \[Smart Contract - Insight] wrong implementation in gt and lt func

Submitted on Mon Jun 17 2024 08:59:38 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32276

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Block stuffing
* Permanent freezing of unclaimed royalties
* Contract fails to deliver promised returns, but doesn't lose value
* Theft of unclaimed yield
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

the IPFSs function can be used to provide a fixed point number, and it contains several functions to provide mul and div and greater or lower than functionality and much more, however the function `gt` and `lt` are not implemented correctly, these function return incorrect boolean when the `self` and `other` have different set of value and this is possible because the function checks for `self` and `self` not between self and `other param`.

## Vulnerability Details

the problem exist in the all `gt && lt` function in the IFP libs, the issue is using the self parameter twice in the body of the function rather than using self and other like shown below:

```sway
//IFP64 lib

    //@audit
    fn gt(self, other: Self) -> bool {
        if self.non_negative && !self.non_negative {
            true
        } else if !self.non_negative && self.non_negative {
            false
        } else if self.non_negative && self.non_negative {
            self.underlying > other.underlying
        } else {
            self.underlying < other.underlying
        }
    }


//@audit same is true for `lt` functions 

 fn lt(self, other: Self) -> bool {
        if self.non_negative && !self.non_negative {
            false
        } else if !self.non_negative && self.non_negative {
            true
        } else if self.non_negative && self.non_negative {
            self.underlying < other.underlying
        } else {
            self.underlying > other.underlying
        }
    }

```

as shown above, we check if `self.non_negative` is true and false at the same time, and this is not correct and its not possible the `self.non_negative` to be true or false at the same time, because of this issue the else if will always run even if the self was negative and other was positive and vice versa. this issue can cause lot of issue when its used by defi apps or third parties(protocols like uniswap or compound or perpetual protocols on ethereum that depends on fixed point number and positive and negative value):

* DOS the contract call if the value was negative when protocol want to handle positive value only.
* incorrect math calculation when doing swap or deposit/borrow and lending mechanism
* attacker can use the incorrect gt and lt function to take more of tokens

keep in mind that this issue exist in all IFP libs.

## Impact Details

incorrect implementation of the ipfs greater than and lower than function can lead to critical issue occur.

## Recommend

change the function impl to something like this:

```sway
 fn gt(self, other: Self) -> bool {
        if self.non_negative && !other.non_negative {
            true
        } else if !self.non_negative && other.non_negative {
            false
        } else if self.non_negative && other.non_negative {
            self.underlying > other.underlying
        } else {
            self.underlying < other.underlying
        }
    }

// same thing can be applied to lt functions too
```

## Proof of concept

## Proof of Concept

RUN : forc test

the POC below can be added in new created fuel project following [this link](https://docs.fuel.network/guides/contract-quickstart/)

```sway

contract;

use sway_libs::fixed_point::{ifp128::IFP128, ifp256::IFP256, ifp64::IFP64,};

abi MyContract {
    fn test_function() -> bool;

}

impl MyContract for Contract {

    fn test_function() -> bool {
   /*
   for num: 

   pub fn from_uint(uint: u32) -> Self {
        Self::from(UFP32::from_uint(uint))
    } 
    which call below
    fn from(value: UFP32) -> Self {
        Self {
            underlying: value,
            non_negative: true,
        }
    }

    for num2:
    pub fn min() -> Self {
        Self {
            underlying: UFP32::min(),
            non_negative: false,
        }
    }

        
        */

        
       let num  = IFP64::from_uint(10u32); // // this will set the non_negative to true
       let num2 = IFP64::min(); // this will set the non_negative to false
      
       assert(num.non_negative() == true);
       assert(num2.non_negative() == false);

       let boolean = IFP64::gt(num, num2); // while this should return false because num and num2 have different value of non)negative, it return true because of the mentioned issue
       
       if boolean == true {
        true

       } else {
        false
       } 

     
    }
   
}

 #[test()]
    fn test_function_contract() {
        let caller = abi(MyContract, CONTRACT_ID);
        let result = caller.test_function {}();
        assert(result == true);
    }

```
