# IOP \_ ThunderNFT 34791 - \[Smart Contract - Low] Incompatibility with SRC might lead to inability of

Submitted on Mon Aug 26 2024 16:33:48 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34791

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/libraries>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `RoyaltyManager` is checking for collection ownership when a user wants to register royalty info for one. The collection's owner is checked against the calling `Identity`. The interface of checking the ownership is not compatible with SRC5 Sway standard and hence is prone to issues which could result in inability to register royalty information.

## Vulnerability Details

Currently the `Ownable` interface is implemented in the following way:

```
abi Ownable {
    #[storage(read)]
    fn owner() -> Option<Identity>;

    #[storage(read)]
    fn admin() -> Option<Identity>;
}
```

however as per the [SRC5 standard](https://docs.fuel.network/docs/sway-standards/src-5-ownership/) the `owner()` method must return `State` enum object of the following definition:

```
pub enum State {
    Uninitialized: (),
    Initialized: Identity,
    Revoked: (),
}
```

which makes it incompatible.

## Impact Details

Currently the verification works, because of how calldata is structured on the contract to contract calls. The calldata verification is loose right now when it comes to types. This however might change with future versions of Fuel and if so it will cause an issue as sellers would not be able to register their royalty information and hence would not be paid royalties.

## Proposed Solution

Change the `ownable_interface.sw` implementation to make it SRC5 compatible:

```
library;

use std::identity::Identity;

pub enum State {
    Uninitialized: (),
    Initialized: Identity,
    Revoked: (),
}

abi Ownable {
    #[storage(read)]
    fn owner() -> State;

    #[storage(read)]
    fn admin() -> Option<Identity>;

    #[storage(read)]
    fn is_admin(identity: Identity) -> bool;
}
```

the `admin` function could be left as is, however as per the current admin standard library implementation it is better to add the `is_admin()` which looks more compatible with the said library: <https://github.com/FuelLabs/sway-libs/blob/master/libs/src/admin.sw>

## References

Ownable interface: <https://github.com/ThunderFuel/smart-contracts/blob/main/contracts-v1/interfaces/src/ownable\\_interface.sw>

## Proof of concept

## Proof of Concept

PoC: <https://gist.github.com/jecikpo/3e3aaf389c9564ffda78974a03d1e9b5>
