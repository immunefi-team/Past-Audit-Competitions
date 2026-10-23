# IOP \_ ThunderNFT 34967 - \[Smart Contract - Insight] Insights Report

## Insights Report

Submitted on Mon Sep 02 2024 00:10:04 GMT-0400 (Atlantic Standard Time) by @Blockian for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34967

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/royalty\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Insight Report

### Description

## Thunder Exchange

### Insights Report

#### Description

This report compiles a series of insights gathered during the review process. While these observations are not eligible for rewards, I think they might be useful for improving the protocol.

For your convenience, I have consolidated them into a single report.

### Redundant use of `only_owner`

In several contract functions such as `transfer_ownership` and `renounce_ownership`, there is a redundancy where the `only_owner` check is called multiple times. For instance, in the `asset_manager` contract:

```rs
    #[storage(read, write)]
    fn transfer_ownership(new_owner: Identity) {
        storage.owner.only_owner();
        storage.owner.transfer_ownership(new_owner);
    }
```

However, the `transfer_ownership` function already includes an `only_owner` check:

```rs
    #[storage(read, write)]
    pub fn transfer_ownership(self, new_owner: Identity) {
        self.only_owner();

        self.write(Ownership::initialized(new_owner));

        log(OwnershipTransferred {
            new_owner,
            previous_owner: msg_sender().unwrap(),
        });
    }
```

This results in unnecessary storage reads and increased gas costs for users. Removing the redundant check would optimize the contract’s performance.

### Royalty Fee Receiver can be Zero Identity

The Royalty Fee Receiver address can be set to zero by any collection registering royalty information. Although this does not have an immediate functional impact, allowing funds to be directed to a zero address is generally undesirable and could lead to future issues. It is recommended to implement a validation check to prevent this scenario.

### Missing `pub` in `ExtraParams`

In `order_types`, the `ExtraParams` struct is currently private, which limits its usability. This restricts users of the protocol who wish to create valid orders by importing the library.

Making the ExtraParams struct public would enhance the protocol’s usability.

### Proof of concept

## POC

Here is a simple POC for setting the Royalty Fee Receiver to Zero Identity

1. First, we need to remove this `else` statement in the `register_royalty_info` function:

```rs
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
        } else { // this else
            // revert(111)
        }
```

We need to remove it for the POC simplicity, otherwise we need to set an Ownable contract and so on. 2. Run this simple POC code:

```rs
contract;

use interfaces::{
  royalty_manager_interface::*,
};

#[test()]
fn test_wrong_log() {
  pub const ZERO_B256 = 0x0000000000000000000000000000000000000000000000000000000000000000;
  pub const ZERO_CONTRACT_ID = ContractId::from(ZERO_B256);

  let royalty_manager = abi(RoyaltyManager, royalty_manager::CONTRACT_ID);
  royalty_manager.initialize();
  royalty_manager.register_royalty_info(ContractId::from(royalty_manager::CONTRACT_ID), Identity::ContractId(ZERO_CONTRACT_ID), 0); // worked fine
}
```
