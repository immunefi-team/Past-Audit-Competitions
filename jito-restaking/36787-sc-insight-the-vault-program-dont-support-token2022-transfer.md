# #36787 \[SC-Insight] The vault program don't support token2022 transfer

**Submitted on Nov 14th 2024 at 12:17:58 UTC by @Hoverfly9132 for** [**Audit Comp | Jito Restaking**](https://immunefi.com/audit-competition/jito-restaking-audit-competition)

* **Report ID:** #36787
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/jito-foundation/restaking/tree/master/vault\\_program>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Bug Description

From the competition page we can know the vault program should support SPL token 2022:

`The Vault and Restaking programs support the SPL Token and SPL Token 2022 standards`.

But when calling `process_mint()` instruction, the token program id is hardcoded to be `spl_token` and not `spl_token_2022`. The spl token and spl token 2022 have different program id, the SPL token program id is `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` and SPL token 2022 program id is `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`, you can see the decalre for [`spl_token program id`](https://github.com/solana-labs/solana-program-library/blob/2d795f15287f6a0907bfe68f3fa252e30ee9572b/token/program/src/lib.rs#L85-L85) and [`spl_token_2022 program id`](https://github.com/solana-labs/solana-program-library/blob/2d795f15287f6a0907bfe68f3fa252e30ee9572b/token/program-2022/src/lib.rs#L95-L95).

```rust
pub fn process_mint(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    amount_in: u64,
    min_amount_out: u64,
) -> ProgramResult {
    ...
    // transfer tokens from depositor to vault
    {
        invoke(
            &transfer(
                // @audit - hardcoded spl token program id
                &spl_token::id(),
                depositor_token_account.key,
                vault_token_account.key,
                depositor.key,
                &[],
                amount_in,
            )?,
            &[
                depositor_token_account.clone(),
                vault_token_account.clone(),
                depositor.clone(),
            ],
        )?;
    }
    ...
}
```

If the users transfer SPL token2022 to the vault by calling the `process_mint()` instruction, the transfer will fail because wrong token program id.

And in the `process_initialize_vault()` instruction, the token program id is loaded by `load_token_program()`, this function will check the token program id is `spl_token` or not, if not, it will return an error:

```rust
pub fn load_token_program(info: &AccountInfo) -> Result<(), ProgramError> {
    if info.key.ne(&spl_token::id()) {
        msg!("Account is not the spl token program");
        return Err(ProgramError::IncorrectProgramId);
    }

    Ok(())
}
```

So the vault program doesn't support SPL token2022 init.

## Impact

The vault program doesn't support SPL token 2022 init and transfer.

## Recommendation

Add the SPL token2022 feature to the vault program.

## Proof of Concept

## Proof of Concept

Place the case in the `vault_program/src/` path, run it by `cargo test --package jito-vault-program --lib -- test_token2022_basic`:

```rust
use solana_program::{
    pubkey::Pubkey,
    system_instruction,
};
use spl_token_2022::{
    instruction::{initialize_mint, initialize_account, mint_to, transfer_checked},
    state::{Mint, Account},
    extension::{ExtensionType, StateWithExtensions},
    ID as TOKEN_2022_ID,
};
use solana_program_test::*;
use solana_sdk::{
    signature::Keypair,
    signer::Signer,
    transaction::Transaction,
};

#[tokio::test]
async fn test_token2022_basic_transfer() {
    let mut program_test = ProgramTest::new(
        "spl_token_2022",
        TOKEN_2022_ID,
        processor!(spl_token_2022::processor::Processor::process),
    );
    program_test.prefer_bpf(false);

    let mut context = program_test.start_with_context().await;
    let payer = &context.payer;
    let mint_authority = Keypair::new();
    let mint = Keypair::new();
    let owner = Keypair::new();
    let recipient = Keypair::new();

    // Create a regular Token-2022 mint
    let mint_size = ExtensionType::try_calculate_account_len::<Mint>(&[]).unwrap();
    let mint_rent = context.banks_client.get_rent().await.unwrap().minimum_balance(mint_size);

    let transaction = Transaction::new_signed_with_payer(
        &[
            system_instruction::create_account(
                &payer.pubkey(),
                &mint.pubkey(),
                mint_rent,
                mint_size as u64,
                &TOKEN_2022_ID,
            ),
            initialize_mint(
                &TOKEN_2022_ID,
                &mint.pubkey(),
                &mint_authority.pubkey(),
                Some(&mint_authority.pubkey()),
                9,
            ).unwrap(),
        ],
        Some(&payer.pubkey()),
        &[payer, &mint],
        context.last_blockhash,
    );
    context.banks_client.process_transaction(transaction).await.unwrap();

    // Create accounts and mint tokens
    let account_size = ExtensionType::try_calculate_account_len::<Account>(&[]).unwrap();
    let rent: u64 = context.banks_client.get_rent().await.unwrap().minimum_balance(account_size);

    // Source account
    let source_account = Keypair::new();
    let transaction = Transaction::new_signed_with_payer(
        &[
            system_instruction::create_account(
                &payer.pubkey(),
                &source_account.pubkey(),
                rent,
                account_size as u64,
                &TOKEN_2022_ID,
            ),
            initialize_account(
                &TOKEN_2022_ID,
                &source_account.pubkey(),
                &mint.pubkey(),
                &owner.pubkey(),
            ).unwrap(),
        ],
        Some(&payer.pubkey()),
        &[payer, &source_account],
        context.last_blockhash,
    );
    context.banks_client.process_transaction(transaction).await.unwrap();

    // Mint tokens
    const MINT_AMOUNT: u64 = 1_000_000;
    let transaction = Transaction::new_signed_with_payer(
        &[mint_to(
            &TOKEN_2022_ID,
            &mint.pubkey(),
            &source_account.pubkey(),
            &mint_authority.pubkey(),
            &[],
            MINT_AMOUNT,
        ).unwrap()],
        Some(&payer.pubkey()),
        &[payer, &mint_authority],
        context.last_blockhash,
    );
    context.banks_client.process_transaction(transaction).await.unwrap();

    // Destination account
    let destination_account = Keypair::new();
    let transaction = Transaction::new_signed_with_payer(
        &[
            system_instruction::create_account(
                &payer.pubkey(),
                &destination_account.pubkey(),
                rent,
                account_size as u64,
                &TOKEN_2022_ID,
            ),
            initialize_account(
                &TOKEN_2022_ID,
                &destination_account.pubkey(),
                &mint.pubkey(),
                &recipient.pubkey(),
            ).unwrap(),
        ],
        Some(&payer.pubkey()),
        &[payer, &destination_account],
        context.last_blockhash,
    );
    context.banks_client.process_transaction(transaction).await.unwrap();

    // Transfer using Token-2022
    const TRANSFER_AMOUNT: u64 = 100_000;
    let transaction = Transaction::new_signed_with_payer(
        &[transfer_checked(
            &TOKEN_2022_ID,
            &source_account.pubkey(),
            &mint.pubkey(),
            &destination_account.pubkey(),
            &owner.pubkey(),
            &[],
            TRANSFER_AMOUNT,
            9,
        ).unwrap()],
        Some(&payer.pubkey()),
        &[payer, &owner],
        context.last_blockhash,
    );
    context.banks_client.process_transaction(transaction).await.unwrap();

    // Try SPL Token transfer (should fail)
    let spl_transfer = Transaction::new_signed_with_payer(
        &[spl_token::instruction::transfer(
            &spl_token::id(),
            &source_account.pubkey(),
            &destination_account.pubkey(),
            &owner.pubkey(),
            &[],
            TRANSFER_AMOUNT,
        ).unwrap()],
        Some(&payer.pubkey()),
        &[payer, &owner],
        context.last_blockhash,
    );
    let result = context.banks_client.process_transaction(spl_transfer).await;
    assert!(result.is_err());

    // Verify balances
    let source = context.banks_client.get_account(source_account.pubkey()).await.unwrap().unwrap();
    let destination = context.banks_client.get_account(destination_account.pubkey()).await.unwrap().unwrap();

    let source_token = StateWithExtensions::<Account>::unpack(&source.data).unwrap();
    let destination_token = StateWithExtensions::<Account>::unpack(&destination.data).unwrap();

    assert_eq!(source_token.base.amount, MINT_AMOUNT - TRANSFER_AMOUNT);
    assert_eq!(destination_token.base.amount, TRANSFER_AMOUNT);
} 
```
