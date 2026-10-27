# Movement Labs Attackathon

## Reports by Severity

<details>

<summary>Critical</summary>

* \#43014 \[BC-Critical] finite Deadlock of Transactions (No Automatic Timeout + Sequential Execution) on multisig implementation
* \#43108 \[BC-Critical] Attackers can front-run transactions in celestia mempool to cause transactions of many users revert unexpectedly
* \#43110 \[BC-Critical] Validator can DoS the DA Layer by requesting a big range of blobs
* \#41987 \[BC-Critical] Oversized blocks split the chain
* \#42143 \[BC-Critical] Decompressing a maliciously crafted blob leads to shutting down all Movement DA Light Nodes in a Movement based network which using a centralized Sequencer.
* \#42749 \[BC-Critical] Attacker can send digests directly to Celestia to reorder block execution
* \#43214 \[BC-Critical] Unchecked transaction size allows malicious users to DOS honest users transactions
* \#42153 \[BC-Critical] Attackers can exploit bug in Blob Verification to execute replay attack by re-executing blobs
* \#43246 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the maptos-opt-executor service
* \#43253 \[BC-Critical] Attackers can drain TIA from nodes in networks running in sequencer mode
* \#43243 \[BC-Critical] Attacker can halt chains operating in sequencer mode
* \#43244 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the Light Node Service
* \#42233 \[BC-Critical] Critical DoS Vulnerability in Movement Network's DA Layer Due to Zstd bomb blob exploit
* \#43288 \[BC-Critical] Attackers could force Nodes to process TraAttackers could force Nodes to process Transactions in wrong order, by attacking moveRocks/sequencing implementation
* \#43290 \[BC-Critical] Anyone can send a write\_batch to the DA node, enabling a DOS attack that shuts down the network
* \#42837 \[BC-Critical] Total network shutdown
* \#43315 \[BC-Critical] DA Light Node Can Be DoSed Due to Lack of Batch Validation
* \#42298 \[BC-Critical] Blocks from Celestia are not executed in order which breaks sequencer logic and application priorities
* \#42941 \[BC-Critical] \[Critical] Network-Wide Denial of Service Through Unrecoverable Block Execution Failures
* \#41531 \[BC-Critical] Attackers can drain the sequencer’s wallet and DoS network by submitting transactions from unfunded accounts
* \#43250 \[BC-Critical] Excessive TCP timeout allows attacker to crash the sequencer via the indexer service
* \#43251 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the finality viewer service
* \#43190 \[BC-Critical] Deadlock in \`submit\_transaction()\`
* \#43330 \[BC-Critical] Freezing new transaction processing by sending invalid requests to movement DA light node
* \#41334 \[BC-Critical] Attacker can publish a blob that cannot be deserialized and shut down the movement chain
* \#43177 \[BC-Critical] DoS Vulnerability in DA Light Node via Unbounded Height Parameter
* \#41489 \[BC-Critical] Blob sizes remain unchecked leading to chain halt
* \#43114 \[BC-Critical] Attackers can cause total shutdown network by exploiting missing of blob size check in DA Lightnode
* \#42112 \[BC-Critical] Using \`blob.GetAll\` instead of \`blob.Get\` for Celestia DA opens full nodes to fraudulent block attacks
* \#42936 \[BC-Critical] Potential Deadlock or Panic Due to Concurrent Lock Acquisition in \`TransactionPipe\`
* \#43333 \[BC-Critical] Missing Depths Checks in Cached TypeLayout leads to Network Divergence
* \#41012 \[BC-Critical] Unintended Chain Split in Movement Full Node

</details>

<details>

<summary>High</summary>

* \#41794 \[BC-High] Not having any whitelisted account completely disables the prevalidator leading to transactions that cannot be deserialized
* \#41686 \[BC-High] The passthrough DA light node streams transactions instead of blocks which means that the block cannot be deserialized
* \#41878 \[BC-High] Edge-case allows replaying user transactions to fill the mempool
* \#43322 \[BC-High] Inadequate Transaction Validation in DA Light Node Allows Unprocessable Block Creation
* \#43054 \[BC-High] Malicious Light Node can DoS the Full Node
* \#43191 \[BC-High] DOS attack by sending transactions that pass the sufficient balance test when entering mempool but fail it in execution
* \#42761 \[BC-High] Memseq does not verify client-specified expiration for transactions before including them in DA (Data Availability).
* \#43241 \[BC-High] Attackers can drain TIA from nodes in networks running in passthrough mode
* \#43307 \[BC-High] Not verifying the signatures upon execution leads to direct loss of funds
* \#42011 \[BC-High] Duplicate tx IDs in blockchain blocks are possible
* \#43323 \[BC-High] Inadequate Sequence Number Validation in DA Light Node Enables Transaction Censorship
* \#43324 \[BC-High] Insufficient Validation in DA Light Node Allows Malicious Override of \`application\_priority\`
* \#41373 \[BC-High] Premature transaction acceptance to mempool/DA without signature validation
* \#42395 \[BC-High] Movement does not allow overwriting transactions with a higher priority, breaking Aptos mempool logic
* \#42903 \[BC-High] Attackers are able to submit multiple dupplicate transactions due to mismatched Mempool Implementation
* \#41516 \[BC-High] The attacker exceeds the number of transactions TOO\_NEW\_TOLERANCE and performs a DoS attack.
* \#41518 \[BC-High] The transaction to modify the gas price was not processed.
* \#42535 \[BC-High] Garbage collecting in flight transactions can lead to spiraling network delays
* \#41368 \[BC-High] JSON-RPC no batch query limitation allows really large responses | RPC server takedown
* \#42991 \[BC-High] User can reuse sequence number causing DOS & breaking core invariant
* \#41715 \[BC-High] Manipulating the Sequence Number of signed transactions to reorder them or prevent their execution
* \#41714 \[BC-High] Tampering the ID of signed transactions to prevent others from executing
* \#42762 \[BC-High] New accounts break the pipe mempool invariant that prevents duplicate transactions from filling the mempool
* \#43017 \[BC-High] Prevalidation does not validate application priority, sequence number and ID
* \#42930 \[BC-High] Users are unable to increase their gas resulting in stuck funds
* \#42896 \[BC-High] Attackers can exploit sequence\_number tolerance mechanism to to cause Movement Network DA Lightnode loose money for submitting failed blocks to Celestia
* \#43150 \[BC-High] Excessive transaction processing caused by a faulty garbage collector in transaction\_pipe.rs
* \#43136 \[BC-High] Multiple transactions sent by the same account in the same block timeframe can get stuck in the TranactionPipe core\_mempool
* \#43135 \[BC-High] \`epilogue\_gas\_payer\` Silently Drops Excess Storage Fee Refunds Under Governed Gas Pool
* \#43222 \[BC-High] A transaction with sequence number 0 can be submitted multiple times
* \#42648 \[BC-High] Altering the application\_priority to fill a block, temporary freezing user transactions
* \#41722 \[BC-High] The passthrough DA light node does not prevalidate transactions which leads to non-deserializable transactions that prevent execution
* \#42934 \[BC-High] Improper input validation in KeylessSignature causes full-node panic
* \#43229 \[BC-High] There is a bug can allows malicious data to enter the DA layer and be signed by a legitimate node
* \#42513 \[BC-High] Users might loose Storage Gas Fee Refund Due to Governed Gas Pool Feature of Movement logic bug
* \#42495 \[BC-High] The Tonic Request/Response Size Limit prevents data from being submitted to the da\_light\_node
* \#42102 \[BC-High] uncontrolled resource consumption is resulting in OOM via RPC (public one)
* \#41437 \[BC-High] An edge-case allows duplicate transactions to be added to the mempool of the sequencer

</details>

<details>

<summary>Medium</summary>

* \#42940 \[BC-Medium] Suboptimal Lock Holding During Logging in \`decrement\_transactions\_in\_flight\`
* \#41678 \[BC-Medium] Transactions directly sent to the passthrough will cause the mempool to accept more transactions than the \`inflight\_limit\`
* \#41864 \[BC-Medium] When Memseq selects a transaction from a particular user to include in a block, it does not remove transactions from Memseq that have a sequence\_number less than or equal to the t...
* \#43312 \[BC-Medium] get\_state\_proof() is called with the current version leading to the epoch\_changes of the StateProof always being empty
* \#42933 \[BC-Medium] Integer Underflow in Garbage Collection Logic of UsedSequenceNumberPool disrupting transaction processing
* \#43148 \[BC-Medium] Potential unhandled panic in protocol-units::execution::maptos::opt-executor::executor/mod::decrement\_transactions\_in\_flight
* \#42480 \[BC-Medium] Unable to deposit the gas fee into the \`governed\_gas\_pool\` when using \`deposit\_from\_fungible\_store\`
* \#41466 \[BC-Medium] Incorrect sequence number tracking in mempool commit
* \#42928 \[BC-Medium] Depositing gas fees into the governed gas pool does not work when the CoinStore is frozen
* \#43137 \[BC-Medium] Multiple Transactions from the same account with increasing sequence number and priorities will be sorted incorrectly in the block causing some to fail
* \#43255 \[BC-Medium] User Transactions might be lost due to missing Error Handling in Celestia RPC Client Requests \`blob\_submit\` failure
* \#43132 \[BC-Medium] upgrade\_burn\_percentage Resets Block Proposer, Blocking Fee Distribution
* \#41255 \[BC-Medium] Blocking sleep in async context leads to thread pool exhaustion and DoS
* \#41669 \[BC-Medium] Incorrect Gas Cost Used for BLS12381 Subgroup Check Causes \~70% Undercharge
* \#43303 \[BC-Medium] The call to \`commit\_transaction()\` includes the wrong sequence number

</details>

<details>

<summary>Low</summary>

* \#43287 \[BC-Low] Certain fees are unaccounted for causing failed transactions
* \#42557 \[BC-Low] Remote signing methods can fail which will turn off the light node block proposer

</details>

<details>

<summary>Insight</summary>

* \#41594 \[BC-Insight] Invalid URL format in TcpListener binding prevents REST API from starting
* \#43326 \[BC-Insight] Stale Transaction State in Mempool When Sender/Receiver Pipe Fails
* \#43187 \[BC-Insight] Movement Full Node Panics and Crashes Uncleanly on Connection failure with DA Light Node
* \#41023 \[BC-Insight] Incomplete transaction decrementing leading to undesired behaviour
* \#41978 \[BC-Insight] Values of the current gc\_slot can be garbage collected in edge case
* \#42938 \[BC-Insight] Inefficient Garbage Collection Implementation in \`UsedSequenceNumberPool\`
* \#43217 \[BC-Insight] Incorrect public key notification after key rotation
* \#43220 \[BC-Insight] The GC\_INTERVAL might not be fitting for the configured sequence\_number\_ttl\_ms
* \#41235 \[BC-Insight] Incorrect celestia bridge keyring flag causes network partition in data availability layer
* \#41243 \[BC-Insight] The mempool garbage collector doesn't fully execute garbage collection on each iteration
* \#42234 \[BC-Insight] Missing Match Arm in to\_single\_key\_authenticators() Allows WebAuthn Signatures Despite WEBAUTHN\_SIGNATURE Being Disabled
* \#41811 \[BC-Insight] Configuration data loss in configfile's \`try\_set\_with\_guard\` due to missing file cursor reset
* \#41731 \[BC-Insight] Race Condition in try\_to\_sign can lead to unverifiable blocks and/or blobs
* \#41618 \[BC-Insight] Timestamp unit doesn't match in GcCounter which causes premature transaction eviction
* \#41324 \[BC-Insight] Celestia auth tokens can be stolen by sniffing websocket requests
* \#43346 \[BC-Insight] Transactions arriving at the node out of sequence order will be rejected due to the has\_invalid\_sequence\_number function
* \#42925 \[BC-Insight] Transactions won't be included on Celestia when the gas price is high, and the transactions on Movement will be forgotten
* \#42430 \[BC-Insight] \`add\_mempool\_transaction()\` does not check if the transaction already exist in the mempool
* \#41560 \[BC-Insight] BlobType of BlobResponse can never be SequencedBlobBlock
* \#42895 \[BC-Insight] Misuse of error
* \#43168 \[BC-Insight] Under normal usage of the blockchain, transactions will not be persisted
* \#41985 \[BC-Insight] Using the test keyring backend is insecure
* \#41899 \[BC-Insight] NatSpec of several functions in \`ethereum.move\` is wrong
* \#42859 \[BC-Insight] Pub key format mismatch in \`InKnownSignersVerifier\`
* \#43184 \[BC-Insight] Vulnerable \`Secp256k1\` version allows validation of malformed signatures
* \#41945 \[BC-Insight] Optimization in \`to\_eip55\_checksumed\_address()\` in \`aptos\_framework::ethereum::()\` module
* \#43186 \[BC-Insight] Flawed documentation when streaming da blobs leads to confusion
* \#42939 \[BC-Insight] Transaction expiration is not validated correctly in mempool and sequencer
* \#41337 \[BC-Insight] Channel buffer size in block proposer is too low leading to network delays and resource exhaustion
* \#43038 \[BC-Insight] There is a permanent operator lockout came from an unsafe key rotation
* \#41855 \[SC-Insight] User is able to circumvent blocklist check by utilizing Solidity's implementation
* \#41980 \[BC-Insight] Full nodes panic in read-only mode whenever a transaction is sent
* \#42222 \[BC-Insight] Garbage Collector can fail to run in a timely manner if building\_time\_ms is set to a low value
* \#42937 \[BC-Insight] Public Exposure of Validator Signer Private Key in Executor Struct
* \#43267 \[BC-Insight] Potential Indefinite Hang (Denial of Service) in Full Node DA Sync Due to Missing Stream Timeout For Light Node Connection
* \#43221 \[BC-Insight] Expired transactions prevent new submissions due to delayed garbage collection

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#41855 \[SC-Insight] User is able to circumvent blocklist check by utilizing Solidity's implementation

</details>

<details>

<summary>Blockchain/DLT</summary>

* \#42940 \[BC-Medium] Suboptimal Lock Holding During Logging in \`decrement\_transactions\_in\_flight\`
* \#41794 \[BC-High] Not having any whitelisted account completely disables the prevalidator leading to transactions that cannot be deserialized
* \#41678 \[BC-Medium] Transactions directly sent to the passthrough will cause the mempool to accept more transactions than the \`inflight\_limit\`
* \#41686 \[BC-High] The passthrough DA light node streams transactions instead of blocks which means that the block cannot be deserialized
* \#43014 \[BC-Critical] finite Deadlock of Transactions (No Automatic Timeout + Sequential Execution) on multisig implementation
* \#41878 \[BC-High] Edge-case allows replaying user transactions to fill the mempool
* \#41864 \[BC-Medium] When Memseq selects a transaction from a particular user to include in a block, it does not remove transactions from Memseq that have a sequence\_number less than or equal to the t...
* \#41594 \[BC-Insight] Invalid URL format in TcpListener binding prevents REST API from starting
* \#43312 \[BC-Medium] get\_state\_proof() is called with the current version leading to the epoch\_changes of the StateProof always being empty
* \#43326 \[BC-Insight] Stale Transaction State in Mempool When Sender/Receiver Pipe Fails
* \#43108 \[BC-Critical] Attackers can front-run transactions in celestia mempool to cause transactions of many users revert unexpectedly
* \#43322 \[BC-High] Inadequate Transaction Validation in DA Light Node Allows Unprocessable Block Creation
* \#43110 \[BC-Critical] Validator can DoS the DA Layer by requesting a big range of blobs
* \#43187 \[BC-Insight] Movement Full Node Panics and Crashes Uncleanly on Connection failure with DA Light Node
* \#41023 \[BC-Insight] Incomplete transaction decrementing leading to undesired behaviour
* \#41978 \[BC-Insight] Values of the current gc\_slot can be garbage collected in edge case
* \#41987 \[BC-Critical] Oversized blocks split the chain
* \#43054 \[BC-High] Malicious Light Node can DoS the Full Node
* \#42933 \[BC-Medium] Integer Underflow in Garbage Collection Logic of UsedSequenceNumberPool disrupting transaction processing
* \#42938 \[BC-Insight] Inefficient Garbage Collection Implementation in \`UsedSequenceNumberPool\`
* \#43148 \[BC-Medium] Potential unhandled panic in protocol-units::execution::maptos::opt-executor::executor/mod::decrement\_transactions\_in\_flight
* \#42143 \[BC-Critical] Decompressing a maliciously crafted blob leads to shutting down all Movement DA Light Nodes in a Movement based network which using a centralized Sequencer.
* \#42749 \[BC-Critical] Attacker can send digests directly to Celestia to reorder block execution
* \#43191 \[BC-High] DOS attack by sending transactions that pass the sufficient balance test when entering mempool but fail it in execution
* \#43214 \[BC-Critical] Unchecked transaction size allows malicious users to DOS honest users transactions
* \#43217 \[BC-Insight] Incorrect public key notification after key rotation
* \#43220 \[BC-Insight] The GC\_INTERVAL might not be fitting for the configured sequence\_number\_ttl\_ms
* \#42761 \[BC-High] Memseq does not verify client-specified expiration for transactions before including them in DA (Data Availability).
* \#42153 \[BC-Critical] Attackers can exploit bug in Blob Verification to execute replay attack by re-executing blobs
* \#43241 \[BC-High] Attackers can drain TIA from nodes in networks running in passthrough mode
* \#41235 \[BC-Insight] Incorrect celestia bridge keyring flag causes network partition in data availability layer
* \#43246 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the maptos-opt-executor service
* \#43253 \[BC-Critical] Attackers can drain TIA from nodes in networks running in sequencer mode
* \#41243 \[BC-Insight] The mempool garbage collector doesn't fully execute garbage collection on each iteration
* \#43243 \[BC-Critical] Attacker can halt chains operating in sequencer mode
* \#43244 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the Light Node Service
* \#42233 \[BC-Critical] Critical DoS Vulnerability in Movement Network's DA Layer Due to Zstd bomb blob exploit
* \#42234 \[BC-Insight] Missing Match Arm in to\_single\_key\_authenticators() Allows WebAuthn Signatures Despite WEBAUTHN\_SIGNATURE Being Disabled
* \#43287 \[BC-Low] Certain fees are unaccounted for causing failed transactions
* \#43288 \[BC-Critical] Attackers could force Nodes to process TraAttackers could force Nodes to process Transactions in wrong order, by attacking moveRocks/sequencing implementation
* \#43290 \[BC-Critical] Anyone can send a write\_batch to the DA node, enabling a DOS attack that shuts down the network
* \#43307 \[BC-High] Not verifying the signatures upon execution leads to direct loss of funds
* \#41811 \[BC-Insight] Configuration data loss in configfile's \`try\_set\_with\_guard\` due to missing file cursor reset
* \#42011 \[BC-High] Duplicate tx IDs in blockchain blocks are possible
* \#42837 \[BC-Critical] Total network shutdown
* \#43315 \[BC-Critical] DA Light Node Can Be DoSed Due to Lack of Batch Validation
* \#43323 \[BC-High] Inadequate Sequence Number Validation in DA Light Node Enables Transaction Censorship
* \#42298 \[BC-Critical] Blocks from Celestia are not executed in order which breaks sequencer logic and application priorities
* \#43324 \[BC-High] Insufficient Validation in DA Light Node Allows Malicious Override of \`application\_priority\`
* \#41373 \[BC-High] Premature transaction acceptance to mempool/DA without signature validation
* \#42395 \[BC-High] Movement does not allow overwriting transactions with a higher priority, breaking Aptos mempool logic
* \#42903 \[BC-High] Attackers are able to submit multiple dupplicate transactions due to mismatched Mempool Implementation
* \#41516 \[BC-High] The attacker exceeds the number of transactions TOO\_NEW\_TOLERANCE and performs a DoS attack.
* \#41518 \[BC-High] The transaction to modify the gas price was not processed.
* \#41731 \[BC-Insight] Race Condition in try\_to\_sign can lead to unverifiable blocks and/or blobs
* \#42480 \[BC-Medium] Unable to deposit the gas fee into the \`governed\_gas\_pool\` when using \`deposit\_from\_fungible\_store\`
* \#42941 \[BC-Critical] \[Critical] Network-Wide Denial of Service Through Unrecoverable Block Execution Failures
* \#42535 \[BC-High] Garbage collecting in flight transactions can lead to spiraling network delays
* \#41618 \[BC-Insight] Timestamp unit doesn't match in GcCounter which causes premature transaction eviction
* \#41324 \[BC-Insight] Celestia auth tokens can be stolen by sniffing websocket requests
* \#41531 \[BC-Critical] Attackers can drain the sequencer’s wallet and DoS network by submitting transactions from unfunded accounts
* \#43346 \[BC-Insight] Transactions arriving at the node out of sequence order will be rejected due to the has\_invalid\_sequence\_number function
* \#43250 \[BC-Critical] Excessive TCP timeout allows attacker to crash the sequencer via the indexer service
* \#43251 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the finality viewer service
* \#42925 \[BC-Insight] Transactions won't be included on Celestia when the gas price is high, and the transactions on Movement will be forgotten
* \#43190 \[BC-Critical] Deadlock in \`submit\_transaction()\`
* \#43330 \[BC-Critical] Freezing new transaction processing by sending invalid requests to movement DA light node
* \#42430 \[BC-Insight] \`add\_mempool\_transaction()\` does not check if the transaction already exist in the mempool
* \#41368 \[BC-High] JSON-RPC no batch query limitation allows really large responses | RPC server takedown
* \#41334 \[BC-Critical] Attacker can publish a blob that cannot be deserialized and shut down the movement chain
* \#41466 \[BC-Medium] Incorrect sequence number tracking in mempool commit
* \#43177 \[BC-Critical] DoS Vulnerability in DA Light Node via Unbounded Height Parameter
* \#41489 \[BC-Critical] Blob sizes remain unchecked leading to chain halt
* \#42991 \[BC-High] User can reuse sequence number causing DOS & breaking core invariant
* \#41715 \[BC-High] Manipulating the Sequence Number of signed transactions to reorder them or prevent their execution
* \#41714 \[BC-High] Tampering the ID of signed transactions to prevent others from executing
* \#43114 \[BC-Critical] Attackers can cause total shutdown network by exploiting missing of blob size check in DA Lightnode
* \#42762 \[BC-High] New accounts break the pipe mempool invariant that prevents duplicate transactions from filling the mempool
* \#43017 \[BC-High] Prevalidation does not validate application priority, sequence number and ID
* \#42112 \[BC-Critical] Using \`blob.GetAll\` instead of \`blob.Get\` for Celestia DA opens full nodes to fraudulent block attacks
* \#42930 \[BC-High] Users are unable to increase their gas resulting in stuck funds
* \#42936 \[BC-Critical] Potential Deadlock or Panic Due to Concurrent Lock Acquisition in \`TransactionPipe\`
* \#42896 \[BC-High] Attackers can exploit sequence\_number tolerance mechanism to to cause Movement Network DA Lightnode loose money for submitting failed blocks to Celestia
* \#42928 \[BC-Medium] Depositing gas fees into the governed gas pool does not work when the CoinStore is frozen
* \#42557 \[BC-Low] Remote signing methods can fail which will turn off the light node block proposer
* \#43150 \[BC-High] Excessive transaction processing caused by a faulty garbage collector in transaction\_pipe.rs
* \#43137 \[BC-Medium] Multiple Transactions from the same account with increasing sequence number and priorities will be sorted incorrectly in the block causing some to fail
* \#43333 \[BC-Critical] Missing Depths Checks in Cached TypeLayout leads to Network Divergence
* \#43136 \[BC-High] Multiple transactions sent by the same account in the same block timeframe can get stuck in the TranactionPipe core\_mempool
* \#43135 \[BC-High] \`epilogue\_gas\_payer\` Silently Drops Excess Storage Fee Refunds Under Governed Gas Pool
* \#43255 \[BC-Medium] User Transactions might be lost due to missing Error Handling in Celestia RPC Client Requests \`blob\_submit\` failure
* \#43132 \[BC-Medium] upgrade\_burn\_percentage Resets Block Proposer, Blocking Fee Distribution
* \#43222 \[BC-High] A transaction with sequence number 0 can be submitted multiple times
* \#42648 \[BC-High] Altering the application\_priority to fill a block, temporary freezing user transactions
* \#41255 \[BC-Medium] Blocking sleep in async context leads to thread pool exhaustion and DoS
* \#41722 \[BC-High] The passthrough DA light node does not prevalidate transactions which leads to non-deserializable transactions that prevent execution
* \#41669 \[BC-Medium] Incorrect Gas Cost Used for BLS12381 Subgroup Check Causes \~70% Undercharge
* \#43303 \[BC-Medium] The call to \`commit\_transaction()\` includes the wrong sequence number
* \#42934 \[BC-High] Improper input validation in KeylessSignature causes full-node panic
* \#43229 \[BC-High] There is a bug can allows malicious data to enter the DA layer and be signed by a legitimate node
* \#42513 \[BC-High] Users might loose Storage Gas Fee Refund Due to Governed Gas Pool Feature of Movement logic bug
* \#42495 \[BC-High] The Tonic Request/Response Size Limit prevents data from being submitted to the da\_light\_node
* \#41012 \[BC-Critical] Unintended Chain Split in Movement Full Node
* \#41560 \[BC-Insight] BlobType of BlobResponse can never be SequencedBlobBlock
* \#42895 \[BC-Insight] Misuse of error
* \#42102 \[BC-High] uncontrolled resource consumption is resulting in OOM via RPC (public one)
* \#43168 \[BC-Insight] Under normal usage of the blockchain, transactions will not be persisted
* \#41985 \[BC-Insight] Using the test keyring backend is insecure
* \#41899 \[BC-Insight] NatSpec of several functions in \`ethereum.move\` is wrong
* \#42859 \[BC-Insight] Pub key format mismatch in \`InKnownSignersVerifier\`
* \#43184 \[BC-Insight] Vulnerable \`Secp256k1\` version allows validation of malformed signatures
* \#41945 \[BC-Insight] Optimization in \`to\_eip55\_checksumed\_address()\` in \`aptos\_framework::ethereum::()\` module
* \#43186 \[BC-Insight] Flawed documentation when streaming da blobs leads to confusion
* \#42939 \[BC-Insight] Transaction expiration is not validated correctly in mempool and sequencer
* \#41337 \[BC-Insight] Channel buffer size in block proposer is too low leading to network delays and resource exhaustion
* \#43038 \[BC-Insight] There is a permanent operator lockout came from an unsafe key rotation
* \#41437 \[BC-High] An edge-case allows duplicate transactions to be added to the mempool of the sequencer
* \#41980 \[BC-Insight] Full nodes panic in read-only mode whenever a transaction is sent
* \#42222 \[BC-Insight] Garbage Collector can fail to run in a timely manner if building\_time\_ms is set to a low value
* \#42937 \[BC-Insight] Public Exposure of Validator Signer Private Key in Executor Struct
* \#43267 \[BC-Insight] Potential Indefinite Hang (Denial of Service) in Full Node DA Sync Due to Missing Stream Timeout For Light Node Connection
* \#43221 \[BC-Insight] Expired transactions prevent new submissions due to delayed garbage collection

</details>
