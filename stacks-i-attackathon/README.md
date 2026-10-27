# Stacks I Attackathon

## Reports by Severity

<details>

<summary>Critical</summary>

* \#37861 \[BC-Critical] SBTC Signer WSTS implementation allows nonce replays such that a malicious signer can steal all funds
* \#38458 \[BC-Critical] The coordinator can submit empty BTC transactions to drain BTC tokens in the multi-sign wallet

</details>

<details>

<summary>High</summary>

* \#37718 \[BC-High] Key rotations bricks the system due to incorrect \`aggregate\_key\` being used to spend the \`peg UTXO\` when signing a sweep transaction
* \#37811 \[BC-High] Missing length check when parsing \`SignatureShareRequest\` in the signers allows the coordinator to halt other signers, shutting down the network
* \#37814 \[BC-High] Signers can crash other signers by sending an invalid \`DkgPrivateShares\` due to missing check before passing the payload to \`SignerStateMachine::process\`
* \#38582 \[BC-High] The \`BitcoinCoreClient::get\_tx\_info\` does not support coinbase transactions, which may cause sBTC to be attacked by btc miners or sBTC donations to be lost
* \#38392 \[BC-High] Signer can steal STX tokens in multi-sign wallet by setting a high stacks tx fee
* \#38740 \[BC-High] The missing check in Deposits::DepositScriptInputs::parse() permits losing funds by sending them to an invalid principal
* \#38053 \[BC-High] A single signer can continuously prevent signatures from being finalized, halting network operations
* \#38477 \[BC-High] A single signer can abort every attempted signing round by providing an invalid packet once the coordinator requests signature shares
* \#38111 \[BC-High] Attackers can send a very large event in a Stacks block so that the Signer can never get the Stacks event
* \#38398 \[BC-High] Malicious Signers can initiate repeated contract calls to cause the multi-sign wallet to lose tx fee
* \#37479 \[BC-High] A single signer can lock users' funds by not notifying other signers of the executed \`sweep\` transaction
* \#38516 \[BC-High] Signer can censor transactions and halt the network by providing an invalid nonce or too many nonces

</details>

<details>

<summary>Medium</summary>

* \#37777 \[BC-Medium] \`Emily.create\_deposit\` can overwrite any deposit to the Pending state
* \#38133 \[BC-Medium] A rogue Signer can censor any deposit request from being processed and fullfilled on the Stacks blockchain
* \#37384 \[BC-Medium] Attacker can front-run call to emily api with incorrect data, preventing legit user from registering their deposit
* \#38551 \[BC-Medium] A signer can request stacks tx nonces in batches in advance and then DoS other signers' sBTC contract calls
* \#37470 \[BC-Medium] SBTC Signers do not page through pending deposit requests making it trivially easy to block legit deposits by spamming Emily API
* \#38270 \[BC-Medium] A signer can send a large number of junk \`WstsNetMessage::NonceRequest\` through P2P to make other signers run out of memory
* \#38003 \[BC-Medium] A malicious coordinator calling \`Emily::update\_deposits\` can make the entire Signers network inoperable
* \#37545 \[BC-Medium] Deposits with a lock\_time of 16 cannot be processed

</details>

<details>

<summary>Low</summary>

* \#38605 \[BC-Low] Lack of fee\_rate/last\_fees validation in handle\_bitcoin\_pre\_sign\_request ebables rogue signer to cause financial loss to depositors
* \#38028 \[BC-Low] There is a Partial Network Degradation Due to DynamoDB GSI Throttling Under High Traffic
* \#38460 \[BC-Low] The coordinator can set a higher BTC tx fee than the current network to make users to pay more fees to the BTC miner
* \#37500 \[BC-Low] Blocklist can be circumvented due to incorrect blocking logic in \`request\_decider::can\_accept\_deposit\_request\`

</details>

<details>

<summary>Insight</summary>

* \#38671 \[BC-Insight] Signer key rotation is not possible due to deadlock between submitting key rotation to Stacks and retrieving it
* \#38030 \[BC-Insight] Coordinator can be crashed by signers on DKG
* \#38223 \[BC-Insight] Attackers can disrupt the tag order of gossip messages to bypass signature verification
* \#38690 \[BC-Insight] A malicious coordinator can run multiple DKG coordination in parallel and manipulate their order to break the signers network
* \#38160 \[BC-Insight] Governance calling \`sbtc-registry.update-protocol-contract\` may cause Stacks' events to be ignored by the signer
* \#37530 \[BC-Insight] Deposits can be completely DoSed due to incorrect transaction construction

</details>

## Reports by Type

<details>

<summary>Blockchain/DLT</summary>

* \#37718 \[BC-High] Key rotations bricks the system due to incorrect \`aggregate\_key\` being used to spend the \`peg UTXO\` when signing a sweep transaction
* \#37777 \[BC-Medium] \`Emily.create\_deposit\` can overwrite any deposit to the Pending state
* \#37811 \[BC-High] Missing length check when parsing \`SignatureShareRequest\` in the signers allows the coordinator to halt other signers, shutting down the network
* \#37814 \[BC-High] Signers can crash other signers by sending an invalid \`DkgPrivateShares\` due to missing check before passing the payload to \`SignerStateMachine::process\`
* \#38582 \[BC-High] The \`BitcoinCoreClient::get\_tx\_info\` does not support coinbase transactions, which may cause sBTC to be attacked by btc miners or sBTC donations to be lost
* \#38605 \[BC-Low] Lack of fee\_rate/last\_fees validation in handle\_bitcoin\_pre\_sign\_request ebables rogue signer to cause financial loss to depositors
* \#37861 \[BC-Critical] SBTC Signer WSTS implementation allows nonce replays such that a malicious signer can steal all funds
* \#38392 \[BC-High] Signer can steal STX tokens in multi-sign wallet by setting a high stacks tx fee
* \#38671 \[BC-Insight] Signer key rotation is not possible due to deadlock between submitting key rotation to Stacks and retrieving it
* \#38458 \[BC-Critical] The coordinator can submit empty BTC transactions to drain BTC tokens in the multi-sign wallet
* \#38028 \[BC-Low] There is a Partial Network Degradation Due to DynamoDB GSI Throttling Under High Traffic
* \#38030 \[BC-Insight] Coordinator can be crashed by signers on DKG
* \#38740 \[BC-High] The missing check in Deposits::DepositScriptInputs::parse() permits losing funds by sending them to an invalid principal
* \#38053 \[BC-High] A single signer can continuously prevent signatures from being finalized, halting network operations
* \#38133 \[BC-Medium] A rogue Signer can censor any deposit request from being processed and fullfilled on the Stacks blockchain
* \#37384 \[BC-Medium] Attacker can front-run call to emily api with incorrect data, preventing legit user from registering their deposit
* \#38460 \[BC-Low] The coordinator can set a higher BTC tx fee than the current network to make users to pay more fees to the BTC miner
* \#38477 \[BC-High] A single signer can abort every attempted signing round by providing an invalid packet once the coordinator requests signature shares
* \#38111 \[BC-High] Attackers can send a very large event in a Stacks block so that the Signer can never get the Stacks event
* \#38551 \[BC-Medium] A signer can request stacks tx nonces in batches in advance and then DoS other signers' sBTC contract calls
* \#37470 \[BC-Medium] SBTC Signers do not page through pending deposit requests making it trivially easy to block legit deposits by spamming Emily API
* \#38223 \[BC-Insight] Attackers can disrupt the tag order of gossip messages to bypass signature verification
* \#38270 \[BC-Medium] A signer can send a large number of junk \`WstsNetMessage::NonceRequest\` through P2P to make other signers run out of memory
* \#38690 \[BC-Insight] A malicious coordinator can run multiple DKG coordination in parallel and manipulate their order to break the signers network
* \#37500 \[BC-Low] Blocklist can be circumvented due to incorrect blocking logic in \`request\_decider::can\_accept\_deposit\_request\`
* \#38160 \[BC-Insight] Governance calling \`sbtc-registry.update-protocol-contract\` may cause Stacks' events to be ignored by the signer
* \#37530 \[BC-Insight] Deposits can be completely DoSed due to incorrect transaction construction
* \#38398 \[BC-High] Malicious Signers can initiate repeated contract calls to cause the multi-sign wallet to lose tx fee
* \#37479 \[BC-High] A single signer can lock users' funds by not notifying other signers of the executed \`sweep\` transaction
* \#38003 \[BC-Medium] A malicious coordinator calling \`Emily::update\_deposits\` can make the entire Signers network inoperable
* \#37545 \[BC-Medium] Deposits with a lock\_time of 16 cannot be processed
* \#38516 \[BC-High] Signer can censor transactions and halt the network by providing an invalid nonce or too many nonces

</details>
