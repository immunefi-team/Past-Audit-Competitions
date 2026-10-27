# #37202 \[SC-Insight] some checks can be removed since its not required(best practice report, not an i

**Submitted on Nov 28th 2024 at 18:19:54 UTC by @zeroK for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37202
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

in the function liquidate, there is a check that make sure the borrower trove is in active statue, however this check is not necessary since this check executes in require\_all\_troves\_are\_active when the `internal_batch_liquidate_troves` executed, this is not an issue and can be accepted as best practice or closed by the team( we let the judgment to the teams).

## Vulnerability Details

the function liquidate implemented as below:

```sway

    #[storage(read, write)]
    fn liquidate(
        id: Identity,
        upper_partial_hint: Identity,
        lower_partial_hint: Identity,
    ) {
        require_trove_is_active(id);
        let mut borrowers: Vec<Identity> = Vec::new();
        borrowers.push(id);
        internal_batch_liquidate_troves(borrowers, upper_partial_hint, lower_partial_hint);
    }

#[storage(read)]
fn require_trove_is_active(id: Identity) {
    let trove = storage.troves.get(id).read();
    require(
        trove.status == Status::Active, 
        "TroveManager: Trove is not active",
    );
}
```

as it shown above, it checks if the borrower trove is active and then call to internal\_batch\_liquidate\_troves invoked which checks for the same thing for each borrower(or one borrower in case of one borrower liquidated):

```sway
#[storage(read, write)]
fn internal_batch_liquidate_troves(
    borrowers: Vec<Identity>,
    upper_partial_hint: Identity,
    lower_partial_hint: Identity,
) {
    // Prevent reentrancy
    require(
        storage
            .lock_internal_batch_liquidate_troves
            .read() == false,
        "TroveManager: Internal batch liquidate troves is locked",
    );
    storage.lock_internal_batch_liquidate_troves.write(true);

    // Ensure there are borrowers to liquidate
    require(
        borrowers
            .len() > 0,
        "TroveManager: No borrowers to liquidate",
    );
    //sanity checks 
    require_all_troves_unique(borrowers);
    require_all_troves_are_active(borrowers); //@audit same check 
    require_all_troves_sorted_by_nicr(borrowers); 
...

}


#[storage(read)]
fn require_all_troves_are_active(borrowers: Vec<Identity>) {
    let mut i = 0;
    while i < borrowers.len() {
        require(
            storage
                .troves
                .get(borrowers.get(i).unwrap())
                .read()
                .status == Status::Active,
            "TroveManager: Trove is not active",
        );
        i += 1;
    }
}


```

the check in liquidate can be removed since the check will be executed in the internal function anyway.

## Impact Details

double checking for active borrower is not necessary when liquidating users.

## References

remove the check in the liquidate function(optional)

## Proof of Concept

## Proof of Concept

there is no POC to be created to show best practice reports in this case.
