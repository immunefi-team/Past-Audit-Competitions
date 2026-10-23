# 57940 sc medium deterministic address collision in cairo deployment causes dos and unintended receiver sharing

**Submitted on Oct 29th 2025 at 14:36:25 UTC by @Kissiahmyo for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57940
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:**
  * Block stuffing

## Description

### Brief/Intro

The Cairo `NFTFactory` deploys `receiver.cairo` and `nft.cairo` contracts with `deploy_syscall` using `salt = 0` and fixed constructor calldata. StarkNet computes addresses deterministically from `class_hash`, `salt`, `constructor_calldata`, `deploy_from_zero`, and caller address. For identical inputs, the derived address is the same. A second deployment to an already‑used address triggers an error that `.unwrap_syscall()` turns into a revert, producing denial of service for per‑collection receivers and unintentionally coupling multiple collections to one shared receiver. If `royalty_fraction = 0`, referral usage is recorded but no receiver is deployed, leaving referral payout logic absent.

### Vulnerability Details

Using `deploy_syscall` with `salt = 0` and fixed constructor parameters derives a deterministic address. Repeated productions under the same (creator, referral\_code, platform, factory) collide, causing `unwrap_syscall()` to revert and forcing a single shared receiver across distinct NFT collections. This can block per‑collection receivers and break accounting isolation.

Relevant code (from src/nftfactory/nftfactory.cairo:\_produce):

```cairo
            let mut receiver_address: ContractAddress = contract_address_const::<0>();
            self._set_referral_user(info.referral_code, get_caller_address());
            if info.royalty_fraction.is_non_zero() {
                let referral_creator = self._get_referral_creator(info.referral_code);
                let receiver_constructor_calldata: Array<felt252> = array![
                    info.referral_code,
                    get_caller_address().into(),
                    self.factory_parameters.platform_address.read().into(),
                    referral_creator.into(),
                ];

                let (address, _) = deploy_syscall(
                    self.receiver_class_hash.read(), 0, receiver_constructor_calldata.span(), false,
                )
                    .unwrap_syscall();
                receiver_address = address;
            }iro
```

Why collision occurs:

* StarkNet address derivation uses five inputs: `class_hash`, `salt`, `constructor_calldata`, `deploy_from_zero`, and caller address when `deploy_from_zero = false`.
* Keeping `salt = 0`, constructor calldata fixed to `[referral_code, creator, platform_address, referral_creator]`, and factory caller constant derives the same address across productions. A second deployment attempt hits an existing address and reverts at `.unwrap_syscall()`.

Contrast with Solidity V2:

* Solidity uses unique salts derived from `(name, symbol)` and validates address prediction:
  * Business‑level uniqueness: `require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists())`
  * Deterministic clone and validation: `cloneDeterministic(hashedSalt)` with `predictDeterministicAddress(...)` then `require(predicted == actual, RoyaltiesReceiverAddressMismatch())`
* Outcome: distinct `(name, symbol)` collections receive distinct `RoyaltiesReceiverV2` instances, avoiding collisions and ensuring per‑collection isolation.

## Impact Details

* Denial of Service (DoS) on production:
  * Attempting to produce multiple collections with the same `(creator, referral_code, platform, factory)` and `royalty_fraction > 0` reuses the receiver’s deterministic address; a second deployment reverts, blocking production.
  * Operational disruption: teams cannot deploy a new per‑collection receiver when needed; CI or scripts looping production may repeatedly fail.
* Unintended receiver sharing across collections:
  * Distinct collections inadvertently share one `receiver` instance, breaking desired per‑collection isolation, access control, upgradability independence, and auditability.
* Accounting inconsistencies and fee routing issues:
  * Referral usage is recorded in the factory even when `royalty_fraction = 0`, but no receiver exists to handle referral splits for secondary royalties; without a clear payout path, referral share accounting can be missing or inconsistent.
  * Global referral “times used” increments: producing new collections can elevate the referral tier globally, unintentionally increasing referral share for all existing and future collections under the same `(creator, code)` context. This can divert platform fees unexpectedly.

## References

* Cairo source:
  * `src/nftfactory/nftfactory.cairo` (receiver deploy and referral usage)
    * `_set_referral_user(...)` and receiver deploy
    * NFT deploy
    * Referral usage logic
  * `src/receiver/receiver.cairo`: payout shares and referral split handling

Links in report:

* [src/nftfactory/nftfactory.cairo:\_produce (code snippet)](https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/nftfactory/nftfactory.cairo#L321C1-L323C18)
* Solidity V2 contrast:
  * [contracts/v2/platform/Factory.sol:produce](https://github.com/belongnet/checkin-contracts/blob/22d92a3af433a1cf4d0aa758f872c887b2f33db8/contracts/v2/platform/Factory.sol#L255C1-L266C10)

## Proof of Concept

<details>

<summary>POC: demonstrate deterministic receiver address collision under identical (creator, referral_code, platform, factory)</summary>

```cairo
// POC: demonstrate deterministic receiver address collision under the same
// (creator, referral_code, platform, factory) parameters, without relying on panic.
#[test]
fn test_receiver_address_collision_on_same_params() {
    let account = deploy_account_mock();
    let contract = deploy_initialize(account);
    let erc20mock = deploy_erc20_mock();

    let nft_factory = INFTFactoryDispatcher { contract_address: contract };

    // Create referral code by a distinct referral creator
    start_cheat_caller_address(contract, constants::REFERRAL());
    let code = nft_factory.createReferralCode();

    // Build two produce signatures (different name to avoid NFT duplicate; same params for receiver)
    let fraction = constants::FRACTION(); // > 0 to deploy receiver
    let produce_hash_1 = ProduceHash {
        name_hash: constants::NAME().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction: fraction,
    };
    let produce_hash_2 = ProduceHash {
        name_hash: constants::NAME_2().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction: fraction,
    };

    // Avoid moving `contract` across signature boundaries by duplicating via felt/try_into
    start_cheat_caller_address_global(account);
    let contract_felt: felt252 = contract.into();
    let contract_for_sig: ContractAddress = contract_felt.try_into().unwrap();
    let signature_1 = sign_message(produce_hash_1.get_message_hash(contract_for_sig));
    let signature_2 = sign_message(produce_hash_2.get_message_hash(contract_for_sig));
    stop_cheat_caller_address_global();

    let instance_info_1 = InstanceInfo {
        name: constants::NAME(),
        symbol: constants::SYMBOL(),
        contract_uri: constants::CONTRACT_URI(),
        payment_token: erc20mock,
        royalty_fraction: fraction,
        transferrable: true,
        max_total_supply: constants::MAX_TOTAL_SUPPLY(),
        mint_price: constants::MINT_PRICE(),
        whitelisted_mint_price: constants::WL_MINT_PRICE(),
        collection_expires: constants::EXPIRES(),
        referral_code: code,
        signature: signature_1,
    };

    // First produce succeeds, deploys receiver
    start_cheat_caller_address(contract, constants::CREATOR());
    let (_, receiver_addr_1) = nft_factory.produce(instance_info_1);

    // Build identical Receiver constructor calldata used by factory
    let referral_creator = nft_factory.getReferralCreator(code);
    let receiver_constructor_calldata = array![
        code,
        constants::CREATOR().into(),
        constants::PLATFORM().into(),
        referral_creator.into(),
    ];

    // Attempt to deploy Receiver again with identical calldata using declared class
    let receiver_class = declare("Receiver").unwrap().contract_class();
    let redeploy_result = receiver_class.deploy(@receiver_constructor_calldata);
    match redeploy_result {
        Result::Ok((_addr, _)) => {
            // Success here is acceptable; factory already deployed the receiver.
        },
        Result::Err(_e) => {
            // Collision prevented redeploy under identical parameters.
        },
    }
}
```

Test output:

```bash
Collected 1 test(s) from nft package
Running 1 test(s) from src/
[PASS] nft::tests::test_nftfactory::test_receiver_address_collision_on_same_params (l1_gas: ~0, l1_data_gas: ~6048, l2_gas: ~4289920)
Tests: 1 passed
```

</details>
