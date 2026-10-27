# #36922 \[SC-Insight] the function claim\_collateral in borrowOperation have read only attribute while

**Submitted on Nov 19th 2024 at 18:44:45 UTC by @zeroK for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #36922
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the function `borrowOperation#claim_collateral` is set to read only attribute, this mean that the function have read access to storage only, but the function invoke calls to the `corSurplus#claim_collateral` function which have write storage attribute, this is issue according to fuel blockchain documents since impure functions which call another impure functions should have same storage attribute to avoid compiler-time error:

`Impure functions which call other impure functions must have at least the same storage privileges or a superset of those for the function called. For example, to call a function with write access a caller must also have write access, or both read and write access. To call a function with read and write access the caller must also have both privileges`

<https://docs.fuel.network/docs/sway/blockchain-development/purity/#purity>

## Vulnerability Details

the `borrowOperation#claim_collateral` implemented as below:

```sway

    // Claim collateral from liquidations
    #[storage(read)]
    fn claim_collateral(asset: AssetId) {
        let coll_surplus = abi(CollSurplusPool, storage.coll_surplus_pool_contract.read().bits());
        coll_surplus.claim_coll(msg_sender().unwrap(), asset);
    }

```

as shown above, the claim\_collateral have read only attribute while the coll\_surplus.claim\_coll have write,read storage attribute which indeed access the storage and write data into it:

```sway
    #[storage(read, write)]
    fn claim_coll(account: Identity, asset: AssetId) { 
        require_is_borrow_operations_contract();
        require_is_valid_asset_id(asset);  
        let balance = storage.balances.get((account, asset)).try_read().unwrap_or(0);

        if balance > 0 {
            storage.balances.insert((account, asset), 0);
            let asset_amount = storage.asset_amount.get(asset).read();
            storage.asset_amount.insert(asset, asset_amount - balance);
            transfer(account, asset, balance);
        }
    }

```

we talked with fuel team in similar issue in specific protocol and the approved that the docs is correct and impure functions should have same storage attribute to avoid any compiler errors.

## Impact Details

claim\_coll with read attribute should have write attribute similar to the function that get invoked inside it.

## References

change the function to below:

```sway
   #[storage(write ,read)]
    fn claim_collateral(asset: AssetId) {
        let coll_surplus = abi(CollSurplusPool, storage.coll_surplus_pool_contract.read().bits());
        coll_surplus.claim_coll(msg_sender().unwrap(), asset);
    }
```

## Proof of Concept

## Proof of Concept

create new project to run the function call below which reverts during compile time:

```sway

contract;
 
abi ContractA {
    #[storage(read)]  //NOTICE: change this to read and write in abi and impl the the test won't revert
    fn receive() -> u64;
}
 
impl ContractA for Contract {
    #[storage(read)]
    fn receive() -> u64 {
 
        return return_45();
    }
}

#[storage(read, write)] // call revert here because of pure calling impure
fn return_45() -> u64 {
  45
}

``
```
