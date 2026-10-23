# IOP \_ ThunderNFT 34702 - \[Smart Contract - Low] the function register\_royalty\_info does not allow to

Submitted on Wed Aug 21 2024 14:42:02 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34702

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/royalty\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the function `register_royalty_info` meant to allow the caller to be the owner of the collection OR the admin, however this is not how the function work when a NFT collection have both owner and admin, this is because the register\_royalty\_info function first check if the contract have owner or not and if it does then it check the caller to be equal to the owner, this will prevent any call came from admin since the owner exist. this is not what the function intended to return.

## Vulnerability Details

the function register\_royalty\_info implemented as below:

```sway

    /// Stores royalty info by admin or owner of the NFT collection contract
    #[storage(read, write)]
    fn register_royalty_info(
        collection: ContractId,
        receiver: Identity,
        fee: u64
    ) {
        let ownable = abi(Ownable, collection.into());

        if (ownable.owner().is_some()) {
            let caller = msg_sender().unwrap();
            let collection_owner = ownable.owner().unwrap();
            require(caller == collection_owner, RoyaltyManagerErrors::CallerMustBeOwnerOrAdmin);
        } else if (ownable.admin().is_some()) {
            let caller = msg_sender().unwrap();
            let collection_admin = ownable.admin().unwrap();
            require(caller == collection_admin, RoyaltyManagerErrors::CallerMustBeOwnerOrAdmin);
        } else {
            revert(111)
        }

        require(fee <= storage.fee_limit.read(), RoyaltyManagerErrors::FeeHigherThanLimit);

        let info = RoyaltyInfo {
            collection: collection,
            receiver: receiver,
            fee: fee
        };

        let option_info: Option<RoyaltyInfo> = Option::Some(info);
        storage.royalty_info.insert(collection, option_info);

        log(RoyaltyRegistryEvent {
            royalty_info: info
        });
    }

```

as shown, the function first before all it checks if the collection have owner, and it did then it check the caller to be the owner which this not always the case since the function mentioned that call by admin is accepted too.

* when admin call the register\_royalty\_info function and the owner exist to the check `if (ownable.owner().is_some())`
* since the `caller` is not the owner, the admin call will always revert.

this won't cause any lose of funds but it leads to `contract failed to return the promised value` impact.

## Impact Details

the function `register_royalty_info` does not allow calls from admin when the owner exist.

## References

its recommended to change the check in register\_royalty\_info to something like below:

```sway
    /// Stores royalty info by admin or owner of the NFT collection contract
    #[storage(read, write)]
    fn register_royalty_info(
        collection: ContractId,
        receiver: Identity,
        fee: u64
    ) {
        let ownable = abi(Ownable, collection.into());

        if(ownable.owner().is_some() || ownable.admin().is_some() ){
            let caller = msg_sender().unwrap(); 
             let collection_owner = ownable.owner().unwrap();
            let collection_admin = ownable.admin().unwrap();


            require(caller == collection_owner || caller == collection_admin);
        }
         else {
            revert(111)
        }

        require(fee <= storage.fee_limit.read(), RoyaltyManagerErrors::FeeHigherThanLimit);

        let info = RoyaltyInfo {
            collection: collection,
            receiver: receiver,
            fee: fee
        };

        let option_info: Option<RoyaltyInfo> = Option::Some(info);
        storage.royalty_info.insert(collection, option_info);

        log(RoyaltyRegistryEvent {
            royalty_info: info
        });
    }


```

## Proof of concept

## Proof of Concept

create new sway project and run forc test with adding sway standards & sway libs in the forc.toml:

```sway
contract;

use sway_libs::{admin::*, ownership::*};
use standards::src5::{SRC5, State};

//this contract shows that contract can have both admin and owner in the same time

abi AdminOwnerContract {
    #[storage(read, write)]
    fn add_admin(new_admin: Identity);
    #[storage(read, write)]
    fn remove_admin(old_admin: Identity);
    #[storage(read)]
    fn is_admin(admin: Identity) -> bool;
    #[storage(read)]
    fn only_admin();
    #[storage(read)]
    fn only_owner_or_admin();
    #[storage(read)]
    fn only_owner();
    #[storage(read, write)]
    fn renounce_ownership();
    #[storage(read, write)]
    fn set_ownership(new_owner: Identity);
    #[storage(read, write)]
    fn transfer_ownership(new_owner: Identity);
    #[storage(read)]  
    fn owner_execute_first() -> bool;
}

impl AdminOwnerContract for Contract {
    #[storage(read, write)]
    fn add_admin(new_admin: Identity) {
        add_admin(new_admin); //this won't work when you call it in sway test since sway does not allow vm.prank similar to foundry to set the caller to owner
    }

    #[storage(read, write)]
    fn remove_admin(old_admin: Identity) {
        revoke_admin(old_admin);
    }

    #[storage(read)]
    fn is_admin(admin: Identity) -> bool {
        is_admin(admin)
    }

    #[storage(read)]
    fn only_admin() {
        only_admin();
    }

    #[storage(read)]
    fn only_owner_or_admin() {
        only_owner_or_admin();
    }

    #[storage(read)]
    fn only_owner() {
        only_owner();
    }

    #[storage(read, write)]
    fn renounce_ownership() {
        renounce_ownership();
    }

    #[storage(read, write)]
    fn set_ownership(new_owner: Identity) {
        initialize_ownership(new_owner);
    }

    #[storage(read, write)]
    fn transfer_ownership(new_owner: Identity) {
        transfer_ownership(new_owner);
    }

}

impl SRC5 for Contract {
    #[storage(read)]
    fn owner() -> State {
        _owner()
    }
}



#[test]
 fn test_attacking() {
    let caller = abi(AdminOwnerContract, CONTRACT_ID);

    let owner = Identity::Address(Address::from(0xbebd3baab326f895289ecbd4210cf886ce41952316441ae4cac35f00f0e882a6));

    caller.set_ownership(owner); 

    let s = caller.owner_execute_first();
    assert(s == true);


    /*
    since sway does not allow simulate call coming from owner similar to vm.prank in foundry, below script can be run by the owner:

    fn main() {
    let x = abi(AdminOwnerContract, 0x79fa8779bed2f36c3581d01c79df8da45eee09fac1fd76a5a656e16326317ef0); // any address to set to the AdminOwnerContract
    let admin = Identity::Address(Address::from(0xCebd3baab326f895289ecbd4210cf886ce41952316441ae4cac35f00f0e882a6));
    x.add_admin(admin);
}
    */


 }
```
