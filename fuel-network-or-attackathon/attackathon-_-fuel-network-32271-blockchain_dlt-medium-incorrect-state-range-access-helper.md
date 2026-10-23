# Attackathon \_ Fuel Network 32271 - \[Blockchain\_DLT - Medium] Incorrect state range access helper

Submitted on Mon Jun 17 2024 08:25:13 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32271

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/FuelLabs/fuel-core/tree/8b1bf02103b8c90ce3ef2ba715214fb452b99885>

Impacts:

* A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

Excessive `key.increase` within loop of contract state range access helper functions may lead to the vm incorrectly erroring on valid storage reads.

## Vulnerability Details

Fuel-vm provides many methods to access contract state. Aside from the normal single slot read and write, instructions such as `SCWQ`, `SRWQ` and SWWQ`provide a convenient interface to read or modify several contiguous slots at once. These instructions rely on`contract\_state\_range`,` contract\_state\_insert\_range`and`contract\_state\_remove\_range\` to work. Unfortunately, all those functions implement in correct checks when performing range access.

The extracted [snippet](https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/storage/src/vm_storage.rs#L328) from `contract_state_range` shows that the `key` is incremented after each slot access. If the `key` ever overflows, the function returns error. However, if the range ends at exactly after the last slot (e.g. accessing 2 slots starting from key `U256::MAX - 1`), the `key.increase` after accessing the `U256::MAX` slot will lead to a faulty error.

```
fn contract_state_range(
    &self,
    contract_id: &ContractId,
    start_key: &Bytes32,
    range: usize,
) -> Result<Vec<Option<Cow<ContractsStateData>>>, Self::DataError> {
    use crate::StorageAsRef;

    let mut key = U256::from_big_endian(start_key.as_ref());
    let mut state_key = Bytes32::zeroed();

    let mut results = Vec::new();
    for _ in 0..range {
        key.to_big_endian(state_key.as_mut());
        let multikey = ContractsStateKey::new(contract_id, &state_key);
        results.push(self.database.storage::<ContractsState>().get(&multikey)?);
        key.increase()?;
    }
    Ok(results)
}
```

## Impact Details

The impact of the bug depends on how contracts using `SCWQ`, `SRWQ` and SWWQ\` are written. In the best case scenario, testing might be able to catch the bug before contract deployment, and developers may change their code to work around it. At the worst case scenario, an un-upgradeable contract that depends on access of storage state ranges might be rendered unusable until the chain client is upgraded to fix the bug.

## References

* `https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/storage/src/vm_storage.rs#L328`
* `https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/storage/src/vm_storage.rs#L346`
* `https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/storage/src/vm_storage.rs#L366`
* `https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/storage/src/vm_storage.rs#L393`

## Proof of concept

## Proof of Concept

Add this test case to `read_sequential_range` in [fuel-core/tests/tests/vm\_storage.rs](https://github.com/FuelLabs/fuel-core/blob/5db3e1997d47438c6995f19cf2da05dd78c29857/tests/tests/vm_storage.rs#L89).

We only provide tests for `contract_state_range`, but all the instances linked in the References section are vulnerable to the same bug.

```
#[test_case(
&[(*u256_to_bytes32(U256::MAX - 1), vec![0; 32]), (*u256_to_bytes32(U256::MAX), vec![0; 32])], *u256_to_bytes32(U256::MAX - 1), 2
=> Ok(vec![Some(vec![0; 32]), Some(vec![0; 32])])
; "range covering last slot incorrectly fails"
)]
```
