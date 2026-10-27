# 57921 sc insight whitelisted role cannot be revoked in nft cairo&#x20;

**Submitted on Oct 29th 2025 at 13:29:09 UTC by @Divine\_Dragon for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57921
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nft/nft.cairo>

Impacts:

* (Not specified in original report)

## Description

### Vulnerability Details

`addWhitelisted` in the `nft.cairo` contract is used by the contract owner to mark a given address as whitelisted. However, there is no function in the contract to revoke a whitelisted status on-chain. Because of this, if the owner needs to revoke whitelist status they must perform changes off-chain (in the back-end). This creates a mismatch between on-chain data and back-end data:

* On-chain: address remains whitelisted (true).
* Back-end: may mark the address as not whitelisted (false) after an off-chain revoke.

This inconsistency can break users' trust: users checking their on-chain status see `true`, but operations like `mintStaticOperation` might rely on back-end whitelist checks and fail because the back-end revoked the status off-chain.

## Mitigation

{% hint style="info" %}
Replace `addWhitelisted` with a toggle-style function (for example `toggleWhitelisted`) that allows the owner to switch an address's whitelisted status between `true` and `false` on-chain. This provides a proper on-chain revoke mechanism and prevents divergence between on-chain and back-end state.
{% endhint %}

## References

<https://github.com/immunefi-team/audit-comp-belong/blob/0cbcde6fd80dbc55a9e3403c8e5a74827dea19e2/src/nft/nft.cairo#L448-L455>

## Proof of Concept

Test demonstrating the inability to re-revoke (only adding is implemented):

```rust
#[test]
#[should_panic(expected: 'Address is already whitelisted')]
fn test_addWhitelisted_whitelisted_already() {
    let contract = deploy();

    let nft = INFTDispatcher { contract_address: contract };

    start_cheat_caller_address(contract, constants::CREATOR());

    nft.addWhitelisted(contract_address_const::<1>());

    // Throws: 'Address is already whitelisted'
    nft.addWhitelisted(contract_address_const::<1>());
}
```

The test indicates once a whitelisted status is set for an address, there is no way to revert that status on-chain using the provided interface.

<details>

<summary>Test run output</summary>

```
 *  Executing task: snforge test nft::tests::test_nft::test_addWhitelisted_whitelisted_already --exact 

   Compiling snforge_scarb_plugin v0.51.1 (git+https://github.com/foundry-rs/starknet-foundry.git?tag=v0.51.1#6f5a02e54c76c2e740c0756568448265e12a6f2d)
    Finished `release` profile [optimized] target(s) in 0.05s
   Compiling test(nft_unittest) nft v0.1.0

    Finished `dev` profile target(s) in 4 seconds

Collected 1 test(s) from nft package
Running 1 test(s) from src/
[PASS] nft::tests::test_nft::test_addWhitelisted_whitelisted_already (l1_gas: ~0, l1_data_gas: ~1344, l2_gas: ~928453)
Tests: 1 passed, 0 failed, 0 ignored, other filtered out
```

</details>
