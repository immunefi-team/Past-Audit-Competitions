# Attackathon \_ Fuel Network 32291 - \[Blockchain\_DLT - Insight] Profiling is incorrect for dependent g

Submitted on Mon Jun 17 2024 12:00:07 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32291

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-vm/tree/0e46d324da460f2db8bcef51920fb9246ac2143b>

Impacts:

* Modification of transaction fees outside of design parameters

## Description

## Brief/Intro

Gas profiling for gas costs that depend on variable input is incorrect for large inputs or small amounts of context gas.

## Vulnerability Details

There are 2 different gas charging cases: fixed and variable cost. The fixed cost variant uses `gas_charge`:

```rs
pub(crate) fn gas_charge(
    cgas: RegMut<CGAS>,
    ggas: RegMut<GGAS>,
    mut profiler: ProfileGas<'_>,
    gas: Word,
) -> SimpleResult<()> {
    profiler.profile(cgas.as_ref(), gas);
    gas_charge_inner(cgas, ggas, gas)
}
```

It can be seen that the profiler is invoked first to track the costs, before the charging happens.

In the variable cost case its the reverse:

```rs
//NOTE: also applies to `dependent_gas_charge`
pub(crate) fn dependent_gas_charge_without_base(
    mut cgas: RegMut<CGAS>,
    ggas: RegMut<GGAS>,
    mut profiler: ProfileGas<'_>,
    gas_cost: DependentCost,
    arg: Word,
) -> SimpleResult<()> {
    let cost =
        dependent_gas_charge_without_base_inner(cgas.as_mut(), ggas, gas_cost, arg)?;
    profiler.profile(cgas.as_ref(), cost);
    Ok(())
}
```

Lets look at the internals of profiling and gas charging:

```rs
pub(crate) fn profile(&mut self, cgas: Reg<CGAS>, gas: Word) {
    ...
    #[cfg(feature = "profile-gas")]
    {
        let gas_use = gas.min(*cgas);
        let location =
            super::current_location(self.current_contract, self.pc, self.is);
        self.profiler.add_gas(location, gas_use);
    }
}

fn gas_charge_inner(
    mut cgas: RegMut<CGAS>,
    mut ggas: RegMut<GGAS>,
    gas: Word,
) -> SimpleResult<()> {
    if *cgas > *ggas {
        Err(Bug::new(BugVariant::GlobalGasLessThanContext).into())
    } else if gas > *cgas {
        ...
        *cgas = 0;

        Err(PanicReason::OutOfGas.into())
    } else {
        *cgas = (*cgas)
            .checked_sub(gas)
            .ok_or_else(|| Bug::new(BugVariant::ContextGasUnderflow))?;
        ...
        Ok(())
    }
}

fn dependent_gas_charge_without_base_inner(
    cgas: RegMut<CGAS>,
    ggas: RegMut<GGAS>,
    gas_cost: DependentCost,
    arg: Word,
) -> Result<Word, PanicOrBug> {
    let cost = gas_cost.resolve_without_base(arg);
    gas_charge_inner(cgas, ggas, cost).map(|_| cost)
}
```

The function `gas_charge_inner` is responsible for decreasing the available gas (context gas, stored in register `cgas`) and is used by both fixed and variable case. As can be seen the function `profile` determines the gas it tracks as the minimum of the gas costs and the context gas (`let gas_use = gas.min(*cgas);`). The profiler assumes that `cgas` has not been deducted from yet and thats why it needs to be invoked first before any deductions happen.

An example were this will go wrong:

* assume cgas = 1000, cost = 900
* after deduction cgas = 100
* profiler tracks min(900, 100) = 100, but should have tracked the cost of 900

## Impact Details

For contexts with large variable cost or relatively little context gas, the gas will be tracked incorrectly. While the effects are not immediate, incorrect profiling is likely to cause inaccurate adjustments of variable costs. That can either lead to a loss for the protocol or overcharging of users

## References

Not applicable

## Proof of concept

## Proof of Concept

add the following test to `profile_gas.rs`:

```rs
#[test]
fn incorrect_gas_tracking() {
    let rng = &mut StdRng::seed_from_u64(2322u64);

    let gas_limit = 1_000;
    let arb_fee_limit = 2_000;
    let maturity = Default::default();
    let height = Default::default();

    // Deploy contract with loops
    let reg_a = 0x20;
    let reg_b = 0x21;

    let script_code = vec![
        op::movi(reg_a, 10),
        op::movi(reg_b, 200000),
        op::aloc(reg_a),
        op::aloc(reg_a),
        op::aloc(reg_b),
        op::ret(RegId::ONE),
    ];

    let tx_deploy =
        TransactionBuilder::script(script_code.into_iter().collect(), vec![])
            .max_fee_limit(arb_fee_limit)
            .add_unsigned_coin_input(
                SecretKey::random(rng),
                rng.gen(),
                arb_fee_limit,
                Default::default(),
                rng.gen(),
            )
            .script_gas_limit(gas_limit)
            .maturity(maturity)
            .finalize_checked(height);

    let output = GasProfiler::default();

    let mut vm = Interpreter::<_, _, _>::with_memory_storage();
    vm.with_profiler(output.clone());
    let mut client = MemoryClient::from_txtor(vm.into());

    let receipts = client.transact(tx_deploy);

    //print all receipts
    for receipt in receipts.iter() {
        println!("{:?}", receipt);
    }

    match output.data() {
        Some(data) => {
            let gas = data.gas();
            for (key, value) in gas.iter() {
                println!("{}: {}", key, value);
            }
        }
        None => {
            panic!("No gas data found");
        }
    }

}
```

This will log:

```
Return { id: 0000000000000000000000000000000000000000000000000000000000000000, val: 1, pc: 10388, is: 10368 }
ScriptResult { result: Success, gas_used: 955 }
Location(script, offset=0): 1
Location(script, offset=4): 1
Location(script, offset=12): 2
Location(script, offset=8): 2
Location(script, offset=16): 58
Location(script, offset=20): 13
```

note that the actual gas\_used is 955, while the sum of the profiler tracked gas is 1+1+2+2+58+13=77, which is off by an order of magnitude.

If you reduce the input value for the last `ALOC` call (by setting `op::movi(reg_b, 100000)` instead of 200000), you will get correct tracking:

```
Return { id: 0000000000000000000000000000000000000000000000000000000000000000, val: 1, pc: 10388, is: 10368 }
ScriptResult { result: Success, gas_used: 488 }
Location(script, offset=0): 1
Location(script, offset=4): 1
Location(script, offset=12): 2
Location(script, offset=8): 2
Location(script, offset=16): 469
Location(script, offset=20): 13
```
