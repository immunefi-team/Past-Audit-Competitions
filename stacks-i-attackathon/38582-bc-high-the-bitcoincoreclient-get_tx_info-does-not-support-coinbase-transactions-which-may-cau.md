# #38582 \[BC-High] The \`BitcoinCoreClient::get\_tx\_info\` does not support coinbase transactions, which

**Submitted on Jan 7th 2025 at 09:31:52 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38582
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Direct loss of funds
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

The `BitcoinCoreClient::get_tx_info` function is used to get tx data from bitcoin RPC and parse it. It will parse the bitcoin RPC response data into a `BitcoinTxInfo` structure.

The `BitcoinTxInfo` structure requires that the bitcoin RPC response data must have a `vin` field. However, the coinbase tx in bitcoin does not contain `vin`. So when parsing the coinbase tx with the `get_tx_info` function, it fails to parse and returns an error.

This may have the following impact:

* Malicious bitcoin miner sets 1 output transfer 1 sat to signers address in coinbase tx each time. Then, sBTC signer will try to parse the coinbase tx and fail, and then all txs in the block will be ignored, and the user's deposits also be ignored.
* Honest bitcoin miners donating BTC to sBTC in coinbase tx will be ignored by sBTC signers.

## Vulnerability Details

The `signer/src/bitcoin/rpc.rs::BitcoinTxInfo` structure is as follows.

```rust
    /// The inputs into the transaction.
    pub vin: Vec<BitcoinTxVin>,
    /// A description of the transactions outputs. This object is missing
    /// the `desc` field in the `scriptPubKey` object. That field is the
    /// "Inferred descriptor for the output".
    pub vout: Vec<BitcoinTxInfoVout>,
```

The `vin` field is not optional. Therefore, if the data of the bitcoin RPC response does not contain `vin`, the `get_tx_info` will fail to parse it.

The `signer/src/block_observer.rs::process_bitcoin_block` code is as follows.

```rust
    async fn process_bitcoin_block(&self, block_header: BitcoinBlockHeader) -> Result<(), Error> {
        let block = self
            .context
            .get_bitcoin_client()
            .get_block(&block_header.hash)
            .await?
            .ok_or(Error::BitcoinCoreMissingBlock(block_header.hash))?;
        let db_block = model::BitcoinBlock::from(&block);

        self.context
            .get_storage_mut()
            .write_bitcoin_block(&db_block)
            .await?;
        self.extract_sbtc_transactions(block_header.hash, &block.txdata)
            .await?;

        tracing::debug!("finished processing bitcoin block");
        Ok(())
    }
```

Note its calling order:

1. call `write_bitcoin_block`
2. call `extract_sbtc_transactions`

It first executes `write_bitcoin_block`, and then executes `extract_sbtc_transactions`. When `extract_sbtc_transactions` fails to parse coinbase tx, the block will no longer be processed.

## Impact Details

1. malicious bitcoin miners can use this bug to make sBTC signers ignore user deposits
2. coinbase tx donations from honest bitcoin miners will be ignored

## References

None

## Proof of Concept

## Proof of Concept

1. Base on: <https://github.com/stacks-network/sbtc/releases/tag/0.0.9-rc4>
2. Run docker

   ```sh
   make devenv-up
   make devenv-down
   ```
3. Get signers bitcoin address

   ```sh
   ./signers.sh info
   ```
4. Enter the `docker-bitcoin-miner-1` docker

   ```sh
   docker exec -it docker-bitcoin-miner-1 bash
   ```
5. Run the following command inside the `docker-bitcoin-miner-1` docker to mine a block to signers bitcoin address

   ```sh
   bitcoin-cli -rpcwallet=main -rpcconnect=bitcoin generatetoaddress 1 "<signers bitcoin address>"
   ```
6. Check the log of *sbtc-signer-1* and you will find that sBTC signer has a parsing error

   ```
   2025-01-07T08:59:09.420166379Z  WARN block-observer:process_bitcoin_block{block_hash=309c10993443ff41a3becb8ba6f3bc8575ef42387754261bb545d95e305964e0}: signer::util: failover client call failed error=failed to retrieve the raw transaction for txid ef66dba196015cd9f4df07da8d45bc2654e8fb116c4e91e18dbb82ed3de56535 from bitcoin-core. JSON-RPC error: JSON decode error: missing field `prevout` at line 1 column 368 retry_num=0 max_retries=1
   ```
