# Attackathon \_ Fuel Network 33351 - \[Smart Contract - Critical] ABI supertraits methods are available

Submitted on Thu Jul 18 2024 13:00:58 GMT-0400 (Atlantic Standard Time) by @cyberthirst for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33351

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

ABI supertraits are, as the docs say, intended to make contract implementations compositional. As such they allow to define methods that can be inherited by the contract implementing the trait. However, what is essential is that these methods shouldn't be available externally as contract methods. However, this is not the case, and the methods are available externally.

## Vulnerability Details

The docs [1](https://docs.fuel.network/docs/sway/advanced/traits/#abi-supertraits) say:

```
Methods in ABIsupertrait are not available externally, i.e. they're not actually contract methods, but they can be used in the actual contract methods
```

Also, in the compiler, we can read [2](https://github.com/FuelLabs/sway/blob/7f8f31843bed8459c53196bd6b689c0b05b05f61/sway-core/src/language/ty/program.rs#L131-L130):

```
// ABI entries are all functions declared in impl_traits on the contract type
// itself, except for ABI supertraits, which do not expose their methods to
// the user
```

Or in [3](https://github.com/FuelLabs/sway/blob/65dbd34e7185a58fc3432397c8a0290fb4d72cc3/sway-core/src/semantic_analysis/ast_node/expression/typed_expression/method_application.rs#L250):

```
                // check if the user calls an ABI supertrait's method (those are private)
                // as a contract method
                if let TypeInfo::ContractCaller { .. } = &*type_engine.get(first_arg.return_type) {
                    return Err(handler.emit_err(
                        CompileError::AbiSupertraitMethodCallAsContractCall {
                            fn_name: method_name.clone(),
                            span,
                        },
                    ));
                }
```

Additionally, suppose that the method `renounce_ownership` comes from a supertrait and we try to call it from a test like this:

```
#[test]
fn test() {
    let caller = abi(MyAbi, CONTRACT_ID);
    caller.renounce_ownership();
}
```

Then we get the following compile error

```
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^ Cannot call ABI supertrait's method as a contract method: "renounce_ownership"
```

If we inspect the build artifacts, namely the contract-abi.json file, we'll see that the file is generated correctly, i.e. the supertrait methods are not in there.

However, if we inspect the IR of the generated contracts, we can see that the super traits methods are available in function dispatch. This is also proved and shown in the PoC.

Also, the compiler test [4](https://github.com/FuelLabs/sway/blob/954a81b626abec7c415b3613dc8222098c410a3d/test/src/e2e_vm_tests/test_programs/should_fail/abi_supertrait_method_call/src/main.sw#L22) tests that the methods aren't available. The test correctly fails, but we think it's exactly because the abi file is generated correctly (but the actual code isn't).

Putting it all together, we established that the supertrait methods shouldn't be available as contract methods. However, as we will show, they actually are.

The cause of the issue should be the following function [5](https://github.com/FuelLabs/sway/blob/ba0d2d7e90babde9f2e495a4432bde71b24ba71a/sway-core/src/language/ty/ast_node.rs#L321):

```
    pub fn contract_fns(&self, engines: &Engines) -> Vec<DeclRefFunction> {
        let mut fns = vec![];

        if let TyAstNodeContent::Declaration(TyDecl::ImplTrait(decl)) = &self.content {
            let decl = engines.de().get(&decl.decl_id);
            if decl.is_impl_contract(engines.te()) {
                for item in &decl.items {
                    if let TyTraitItem::Fn(f) = item {
                        fns.push(f.clone());
                    }
                }
            }
        }

        fns
    }
}
```

The function is called to collect the contract function. As we can see, it matches against `ImplTrait`, and there isn't any further validation.

After the functions are collected, they are passed to `generate_contract_entry` [6](https://github.com/FuelLabs/sway/blob/3993f253fb1d2e8846a7e2813511053aff5c59dd/sway-core/src/semantic_analysis/ast_node/declaration/auto_impl.rs#L565) which is responsible for the dispatch. And therefore the function are made externally available.

## Impact Details

This bug makes supertrait methods externally available, i.e. methods which should be private are publicly available and can be immediately called by an attacker.

This is extremely critical because private methods usually don't employ access controls as they shouldn't be available to external users.

As such, one of the most essential forms of access control is broken.

Because this behavior will almost never be tested by the users of Sway (because this behavior is assumed to be guaranteed by the compiler), it has very low likelihood to be discovered during testing.

## References

## Proof of concept

## Proof of Concept

In the PoC, we should that function `renounce_ownership`, which should be private, is actually public. As such, it can be called by an attacker - which we demonstrate. The end result is that the owner loses the ownership of his contract.

This whole sequence is demonstrated via the following asserts:

```
assert(callerA.owner() == CONTRACT_ID);

let callerB = abi(Test, CONTRACT_ID);

callerB.renounce_ownership();  // ! Externally available

assert(callerA.owner() != CONTRACT_ID);
assert(callerA.owner() == b256::zero());
```

To run the PoC, just unzip and run `forc test`.

Also, if we inspect the `supertraits-poc-abi.json` file and check the contract functions, we get:

```
  "functions": [
    {
      "inputs": [],
      "name": "init",
      "output": {
        "name": "",
        "type": 0,
        "typeArguments": null
      },
      "attributes": [
        {
          "name": "storage",
          "arguments": [
            "read",
            "write"
          ]
        }
      ]
    },
    {
      "inputs": [],
      "name": "owner",
      "output": {
        "name": "",
        "type": 1,
        "typeArguments": null
      },
      "attributes": [
        {
          "name": "storage",
          "arguments": [
            "read"
          ]
        }
      ]
    }
  ],
```

as can be seen, the supertrait methods aren't available.

Link to Google Drive with the PoC: <https://drive.google.com/file/d/1jxEiA55O9XkWiIbYqf0-j9zNy\\_pzmiOw/view?usp=sharing>
