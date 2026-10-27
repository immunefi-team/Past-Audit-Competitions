# 57712 sc medium receiver deployment dos via salt reuse

**Submitted on Oct 28th 2025 at 11:33:50 UTC by @pirex for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57712
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

### Brief/Intro

The NFT factory deploys royalty receiver contracts using a constant salt (`0`), causing deterministic address collisions when the same referral code and creator combination is reused across multiple collections. Once a receiver is deployed for a given set of parameters, any subsequent `produce` call with identical constructor calldata reverts, permanently blocking that creator from launching additional royalty-enabled collections with the same promoter. This creates a denial-of-service condition for legitimate platform operations.

### Vulnerability Details

The factory's receiver deployment logic in `src/nftfactory/nftfactory.cairo:321-338` uses Starknet's `deploy_syscall` with a hardcoded salt value of `0`:

{% code title="Vulnerable snippet (nftfactory.cairo:321-338)" %}

```cairo
let (receiver_address, _) = deploy_syscall(
    receiver_class_hash,
    0,  // constant salt
    receiver_constructor_calldata.span(),
    false
).unwrap();
```

{% endcode %}

The receiver constructor calldata is built from:

```cairo
array![
    referral_code,
    creator.into(),
    platform.into(),
    referral_creator.into()
]
```

Starknet's address derivation is deterministic:

```
address = hash(deployer, salt, class_hash, constructor_calldata)
```

Since `salt` is always `0` and the `class_hash` is constant, any two `produce` calls with the same `referral_code` and `creator` will compute identical constructor calldata, resulting in the same receiver address. The first deployment succeeds, but the second attempt to deploy to an already-occupied address causes `deploy_syscall` to revert.

{% stepper %}
{% step %}

### Realistic scenario — how the DoS occurs

1. Venue "CryptoClub" gets referral code 42
2. Artist Alice launches collection "Summer Vibes" with `referral_code=42`
3. Factory deploys receiver at address 0xAAA
4. Artist Alice wants to launch "Winter Blues" with same `referral_code=42`
5. Factory attempts to deploy receiver at 0xAAA again
6. `deploy_syscall` reverts — Alice cannot launch any more collections with CryptoClub

This is not an edge case: promoters are meant to have persistent referral codes across multiple collections, so this collision occurs during normal usage.
{% endstep %}
{% endstepper %}

### Impact Details

* Business continuity:
  * Creators become permanently locked out of using specific referral codes after their first collection
  * Promoters cannot work with the same creator more than once
  * The platform's core model (recurring creator-promoter relationships) breaks down
* Financial:
  * Creators lose the ability to launch new collections with established promotional partners
  * Promoters lose commission opportunities from repeat collaborations
  * Platform loses listing fees from blocked collections
* Severity justification:
  * No funds are directly stolen, but this is Medium severity because it permanently disrupts a core protocol function (collection creation). The DoS is permanent for the matching creator-promoter pairing.
* Attack vector:
  * No special privileges required. Any user can trigger this by creating two collections with the same referral code; the second `produce` call will fail.

## References

* Vulnerable deployment: `src/nftfactory/nftfactory.cairo:321-338`
* Constructor calldata: `src/nftfactory/nftfactory.cairo:324-329`
* Starknet address derivation: <https://docs.starknet.io/documentation/architecture\\_and\\_concepts/Smart\\_Contracts/contract-address/>

## Proof of Concept

<details>

<summary>Test reproducing the DoS (expand to view)</summary>

The test `test_produce_receiver_address_collision` demonstrates the DoS condition:

{% code title="test\_produce\_receiver\_address\_collision" %}

```cairo
#[test]
fn test_produce_receiver_address_collision() {
    let account = deploy_account_mock();
    let contract = deploy_initialize(account);
    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    // Promoter creates referral code
    start_cheat_caller_address(contract, constants::REFERRAL());
    let referral_code = nft_factory.createReferralCode();
    stop_cheat_caller_address(contract);

    let royalty_fraction = constants::FRACTION();
    let erc20mock = deploy_erc20_mock();

    // First collection deploys successfully
    let first_hash = ProduceHash {
        name_hash: constants::NAME().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction,
    };

    start_cheat_caller_address_global(account);
    let first_signature = sign_message(first_hash.get_message_hash(contract));
    stop_cheat_caller_address_global();

    start_cheat_caller_address(contract, constants::CREATOR());
    let first_instance = InstanceInfo {
        name: constants::NAME(),
        symbol: constants::SYMBOL(),
        contract_uri: constants::CONTRACT_URI(),
        payment_token: erc20mock,
        royalty_fraction,
        transferrable: true,
        max_total_supply: constants::MAX_TOTAL_SUPPLY(),
        mint_price: constants::MINT_PRICE(),
        whitelisted_mint_price: constants::WL_MINT_PRICE(),
        collection_expires: constants::EXPIRES(),
        referral_code,
        signature: first_signature.clone(),
    };

    let (_, receiver_address) = nft_factory.produce(first_instance.clone());

    // Verify deterministic address calculation
    let referral_creator = nft_factory.getReferralCreator(referral_code);
    let receiver_constructor_calldata: Array<felt252> = array![
        referral_code,
        constants::CREATOR().into(),
        constants::PLATFORM().into(),
        referral_creator.into(),
    ];

    let receiver_class = declare("Receiver").unwrap().contract_class();
    let expected_receiver = calculate_contract_address_from_deploy_syscall(
        0, *receiver_class.class_hash, receiver_constructor_calldata.span(), contract,
    );

    assert_eq!(receiver_address, expected_receiver);

    // Second deployment with same params yields same address
    let second_expected_receiver = calculate_contract_address_from_deploy_syscall(
        0, *receiver_class.class_hash, receiver_constructor_calldata.span(), contract,
    );

    assert_eq!(expected_receiver, second_expected_receiver);

    // Attempt second produce call - will revert
    let mut produce_calldata = array![];
    produce_calldata.append_serde(first_instance.clone());

    let second_attempt = call_contract_syscall(
        contract, selector!("produce"), produce_calldata.span(),
    );

    match second_attempt {
        Result::Ok(_) => panic_with_felt252('REUSE_OK'),
        Result::Err(panic_data) => {
            assert(panic_data.len() > 0, 'missing panic data');
        }
    }

    stop_cheat_caller_address(contract);
}
```

{% endcode %}

Run with:

```bash
snforge test test_produce_receiver_address_collision
```

The test confirms:

1. First `produce` succeeds and deploys a receiver at a predictable address
2. Second `produce` with identical parameters attempts to redeploy at the same address
3. `deploy_syscall` reverts with panic data, blocking the collection launch

</details>

## Recommended fix

Derive a unique salt for each receiver deployment so the computed address cannot collide for repeated creator/referral combinations. Example options:

{% code title="Recommended fix — use hashed salt or counter" %}

```cairo
let salt = hash(creator, referral_code, collection_counter);
let (receiver_address, _) = deploy_syscall(
    receiver_class_hash,
    salt,
    receiver_constructor_calldata.span(),
    false
).unwrap();
```

{% endcode %}

Alternatives:

* Use an incrementing per-creator or global counter (e.g., `collection_counter`) as salt.
* Hash the full collection metadata into the salt for uniqueness.

These ensure each receiver deployment yields a distinct address and prevent permanent DoS from repeated referral/creator reuse.
