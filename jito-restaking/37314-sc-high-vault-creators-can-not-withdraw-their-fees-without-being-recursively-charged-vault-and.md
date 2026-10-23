# #37314 \[SC-High] Vault creators can not withdraw their fees without being recursively charged (vault

**Submitted on Dec 2nd 2024 at 05:56:27 UTC by @niroh for** [**Audit Comp | Jito Restaking**](https://immunefi.com/audit-competition/jito-restaking-audit-competition)

* **Report ID:** #37314
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/jito-foundation/restaking/tree/master/vault\\_program>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The following fees are collected in the Jito Restaking system: Vault fees - include withdraw fee, deposit fee and reward fee, set as bps by vault\_fee\_admin and collected as vrt\_tokens during withdrawal/deposit/vault\_update\_balance. Program fee - set as bps by the config\_admin and collected during withdrawals. all fee bps changes can only take effect at the start of the next process\_initialize\_vault\_update\_state\_tracker. vault fees are sent to the vault\_fee\_wallet, and program fees to the program\_fee\_wallet, all in the form of vrt\_tokens.

## Vulnerability Details

The problem is that when a vault creator/admin wishes to withdraw their fees from the fee\_wallet, they are subject to the same fee collection rules of any other withdrawal, including a "recursive" charge of withdrawal\_fee\_bps from the wallet to itself. This causes both a loss through paying excessive program fees, and a DOS on full fee withdrawal, as the following fee withdrawal flow demonstrates:

1. Assume vault\_withdrawal\_fee and program\_fee are 10% each
2. A user withdraws 100\_000 vrts
3. 10\_000 vrts are sent to the vault\_fee\_wallet and 10\_000 vrts are sent to the program\_fee\_wallet, and the user receives the ST value of 80\_000 vrts
4. Next the owner of vault\_fee\_wallet wants to withdraw the 10\_000 vrts it received, and so it enqueues a withdrawal ticket for its 10\_000 vrts.
5. Two epocs later, when the withdrawal ticket matures and burned, the burn\_withdrawal\_ticket instruction will "send back" to the vault\_fee\_wallet 1000 vrts, send 1000 vrts to the program\_fee\_wallet and send the ST value of 8000 vrts to the vault\_fee\_wallet.
6. The owner of vault\_fee\_wallet will now have to enqueue a new withdrawal ticket for the "sent back" fee of 1000 VRTs, which will again, only enable withdrawing 80% of the amount, charging the rest as vault/program fees. For a full withdrawal the vault\_fee\_wallet will have to recursively enqueue withdrawals multiple times until the amount left is so small it will be fully withdrawn. (see POC for detailed example).

### Notes

1. While the vault creator controls the withdrawal\_fee\_bps, they can not resolve this by setting the withdrawal\_fee\_bps to zero before they burn the withdrawal ticket and back up after the burn. The reason is that fee changes only take effect on the next process\_initialize\_vault\_update\_state\_tracker. If the vault owner changes the fee to zero before the update cycle of the epoc in which they burn their withdrawal ticket, this change will remain in effect atleast until the next epoc, causing an unpredictable loss from forgone withdrawal fees during that epoc.
2. The same problem exists for the program fee. When the program\_fee\_wallet owner tries to withdraw the fee, program\_fee\_wallet will recursively "charge to itself" the program fee in addition to paying the vault withdrawal\_fee. Similarly to the vault\_fee\_wallet, full withdrawal will be DOSed for a prologed time due to the multiple calls required, and will cause the program to over-pay vault withdrawal fees.

## Recomendation

3. Make withdrawals signed by the vault\_fee\_wallet/program\_fee\_wallet exempt from fees. They can be either entirely exempt (from both vault and program fees), or each exempt only from recursively paying themselves fees. (depending on the protocol's desired behavior)

## Impact Details

This bug results in two main impacts:

1. loss of funds for the owner of vault\_fee\_wallet, through overpaying program fees (see POC example where 25% higher program fees are paid). While technically not "theft", the impact from the vault owner perspective is the same, as their funds end up in someone else's hands.
2. A secondary impact is DOS (and additional transaction costs) of full fee withdrawal. This is due to the need to perform multiple withdraws that require each a couple of epocs to complete. In the POC example, a full withdrawal of fees will take atleast 42 epocs or 84 days. Considering that during this period new fees are expected to accrue, this practically means the vault owner is never able to fully withdraw fees.

## References

<https://github.com/jito-foundation/restaking/blob/406903e569da657035a2ca71ad16f8a930db6940/vault\\_program/src/burn\\_withdrawal\\_ticket.rs#L94> <https://github.com/jito-foundation/restaking/blob/406903e569da657035a2ca71ad16f8a930db6940/vault\\_core/src/vault.rs#L1008>

## Proof of Concept

## Proof of Concept

### How to run

1. In integration\_tests/tests/fixtures/vault\_client.rs line 236, change the program\_fee\_bps parameter from 0 to 1000 (quick workaround to set a 10% program fee)
2. Copy the code below to a new test file under integration\_tests/tests/vault/
3. Run `RUST_LOG=off RUST_BACKTRACE=1 cargo nextest run --nocapture test_vault_fee_withdrawal`

```rust
#[cfg(test)]
mod tests {
    //use jito_vault_sdk::error::VaultError;
    use solana_sdk::signature::{Keypair, Signer};
    use jito_vault_core::{config::Config};
    use crate::fixtures::fixture::{ConfiguredVault, TestBuilder};
    use spl_associated_token_account::get_associated_token_address;
 
    #[tokio::test]
    async fn test_vault_fee_withdrawal() {
        let mut fixture = TestBuilder::new().await;

        let deposit_fee_bps = 0;
        let withdrawal_fee_bps = 2000;
        let reward_fee_bps = 0;
        let num_operators = 2;
        let slasher_amounts = vec![];

        let (ConfiguredVault {
            mut vault_program_client,
            mut restaking_program_client,
            vault_root,
            operator_roots,
            ..
        }) = fixture
            .setup_vault_with_ncn_and_operators(
                deposit_fee_bps,
                withdrawal_fee_bps,
                reward_fee_bps,
                num_operators,
                &slasher_amounts,
                
            )
            .await
            .unwrap();
        
        //get config data
        let mut config = vault_program_client
            .get_config(&Config::find_program_address(&jito_vault_program::id()).0)
            .await
            .unwrap();

           
        //get vault    
        let mut vault = vault_program_client.get_vault(&vault_root.vault_pubkey).await.unwrap();
        let vault_vrt_fee_wallet = &get_associated_token_address(&vault.fee_wallet, &vault.vrt_mint);
        //create a supported mint account to the vault admin since they are also the fee wallet and need to be able to receive STs
        vault_program_client.create_ata(&vault.supported_mint, &vault_root.vault_admin.pubkey()).await.unwrap();


        //get program fee wallet account address
        let program_vrt_fee_wallet_address = &get_associated_token_address(&config.program_fee_wallet, &vault.vrt_mint);

        //get operator keys
        let operator_pubkeys: Vec<_> = operator_roots
            .iter()
            .map(|root| root.operator_pubkey)
            .collect();
        
        //Warp two epocs 
        let epoch_length = config.epoch_length();        
        fixture.warp_slot_incremental(epoch_length*2).await.unwrap(); 
        
        //create depositor
        let initial_deposit = 100_000 * 10_u64.pow(9); //assuming 9 decimals
        let depositor = Keypair::new();
        vault_program_client
          .configure_depositor(&vault_root, &depositor.pubkey(), initial_deposit)
          .await
         .unwrap();

        //full vault update
        vault_program_client.do_full_vault_update(&vault_root.vault_pubkey,&operator_pubkeys).await.unwrap();

        //make a deposit of 100_000 for depositor
        let result = vault_program_client
        .do_mint_to(&vault_root, &depositor, initial_deposit, 0)
        .await;

        //withdraw depositor
        let depositor_vrt_wallet = &get_associated_token_address(&depositor.pubkey(), &vault.vrt_mint);
        let depositor_vrt_wallet_account = fixture.get_token_account(&depositor_vrt_wallet).await.unwrap();
        println!("Depositor vrt balance: {}",depositor_vrt_wallet_account.amount);
        let ticket_info = vault_program_client
            .do_enqueue_withdrawal(&vault_root, &depositor, depositor_vrt_wallet_account.amount).await.unwrap();

        //Warp two epocs       
        fixture.warp_slot_incremental(epoch_length*2).await.unwrap(); 
        vault_program_client.do_full_vault_update(&vault_root.vault_pubkey,&operator_pubkeys).await.unwrap();


        //burn depositor withdrawal ticket
        //test emptying the depositor lamports to see that the account will be closed
        let _ = vault_program_client.do_burn_withdrawal_ticket(
            &vault_root,
             &depositor, 
            &ticket_info.base, 
            &config.program_fee_wallet).await;

        //check vault_fee_wallet vrt_token balance
        let vault_vrt_fee_account = fixture.get_token_account(&vault_vrt_fee_wallet).await.unwrap();
        println!("Vault fee wallet vrt balance before withdraw loop: {}",vault_vrt_fee_account.amount);



         //loop-withdraw the vrt_tokens in the vault_vrt_fee_wallet until the entire starting fee is withdrawn
        let mut epocs=0;
        let staring_vrt = vault_vrt_fee_account.amount;
        let mut left_vert =staring_vrt;
        let mut program_total_fee_vrt = 0;
        while left_vert > 0 {
            //create vault fee withdrawal ticket
            let v_ticket_info = vault_program_client
            .do_enqueue_withdrawal(&vault_root, &vault_root.vault_admin, left_vert).await.unwrap();


            //Warp two epocs       
            fixture.warp_slot_incremental(epoch_length*2).await.unwrap(); 
            vault_program_client.do_full_vault_update(&vault_root.vault_pubkey,&operator_pubkeys).await.unwrap();
            epocs+=2;

            let program_fee_bal_before = fixture.get_token_account(&program_vrt_fee_wallet_address).await.unwrap().amount;

            //burn vault fee withdrawal ticket
            let result = vault_program_client.do_burn_withdrawal_ticket(
                &vault_root,
                &vault_root.vault_admin, 
                &v_ticket_info.base, 
                &config.program_fee_wallet).await;
            match result {
                Err(e) => {eprintln!(" Error: {:?}", e); }// Log or handle the error},
                Ok(()) => {}
            }  
            let program_fee_bal_after = fixture.get_token_account(&program_vrt_fee_wallet_address).await.unwrap().amount;  
            let curr_program_fee_diff = program_fee_bal_after - program_fee_bal_before;
             //check fee wallet left VRT 
            // let vault_vrt_fee_wallet = &get_associated_token_address(&vault.fee_wallet, &vault.vrt_mint);
            let vault_vrt_fee_account = fixture.get_token_account(&vault_vrt_fee_wallet).await.unwrap();
            println!("Epoc {epocs} Vault fee wallet vrt balance leftover after burn: {}",vault_vrt_fee_account.amount);  
            println!("Epoc {epocs}  program fee gained: {}\n",curr_program_fee_diff);  

            left_vert = vault_vrt_fee_account.amount;
            program_total_fee_vrt+=curr_program_fee_diff;
        }
        let effective_bps = program_total_fee_vrt  * 10000 / staring_vrt;
        println!("Full withdrawal of {staring_vrt} VRTs took {epocs} epocs. total program fee  was {program_total_fee_vrt}\n program fee bps: {} effective program fee bps: {effective_bps}",config.program_fee_bps());
        //Output - Full withdrawal of 20000000000000 VRTs took 42 epocs. total program fee  was 2500000000005
        //program fee bps: 1000 effective program fee bps: 1250
        //The vault pays 25% more fee than defined and takes 42 epocs to fully withdraw

    }


}

```
