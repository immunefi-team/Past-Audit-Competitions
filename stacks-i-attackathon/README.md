# Attackathon Stacks

## Reports by Severity

[Critical](<README.md#critical>) | [High](<README.md#high>) | [Medium](<README.md#medium>) | [Low](<README.md#low>) | [Insight](<README.md#insight>)
<details>

<summary>Critical</summary>

* [37861 - [BC - Critical] SBTC Signer WSTS implementation allows nonce replays such that a malicious signer can steal all funds](./37861-bc-critical-sbtc-signer-wsts-implementation-allows-nonce-replays-such-that-a-malicious-signer.md)
* [38458 - [BC - Critical] The coordinator can submit empty BTC transactions to drain BTC tokens in the multi-sign wallet](./38458-bc-critical-the-coordinator-can-submit-empty-btc-transactions-to-drain-btc-tokens-in-the-multi.md)

</details>

<details>

<summary>High</summary>

* [37718 - [BC - High] Key rotations bricks the system due to incorrect \`aggregate\_key\` being used to spend the \`peg UTXO\` when signing a sweep transaction](./37718-bc-high-key-rotations-bricks-the-system-due-to-incorrect-aggregate_key-being-used-to-spend-the.md)
* [37811 - [BC - High] Missing length check when parsing \`SignatureShareRequest\` in the signers allows the coordinator to halt other signers, shutting down the network](./37811-bc-high-missing-length-check-when-parsing-signaturesharerequest-in-the-signers-allows-the-coor.md)
* [37814 - [BC - High] Signers can crash other signers by sending an invalid \`DkgPrivateShares\` due to missing check before passing the payload to \`SignerStateMachine::process\`](./37814-bc-high-signers-can-crash-other-signers-by-sending-an-invalid-dkgprivateshares-due-to-missing.md)
* [38582 - [BC - High] The \`BitcoinCoreClient::get\_tx\_info\` does not support coinbase transactions, which may cause sBTC to be attacked by btc miners or sBTC donations to be lost](./38582-bc-high-the-bitcoincoreclient-get_tx_info-does-not-support-coinbase-transactions-which-may-cau.md)
* [38392 - [BC - High] Signer can steal STX tokens in multi-sign wallet by setting a high stacks tx fee](./38392-bc-high-signer-can-steal-stx-tokens-in-multi-sign-wallet-by-setting-a-high-stacks-tx-fee.md)
* [38740 - [BC - High] The missing check in Deposits::DepositScriptInputs::parse() permits losing funds by sending them to an invalid principal](./38740-bc-high-the-missing-check-in-deposits-depositscriptinputs-parse-permits-losing-funds-by-sendin.md)
* [38053 - [BC - High] A single signer can continuously prevent signatures from being finalized, halting network operations](./38053-bc-high-a-single-signer-can-continuously-prevent-signatures-from-being-finalized-halting-netwo.md)
* [38477 - [BC - High] A single signer can abort every attempted signing round by providing an invalid packet once the coordinator requests signature shares](./38477-bc-high-a-single-signer-can-abort-every-attempted-signing-round-by-providing-an-invalid-packet.md)
* [38111 - [BC - High] Attackers can send a very large event in a Stacks block so that the Signer can never get the Stacks event](./38111-bc-high-attackers-can-send-a-very-large-event-in-a-stacks-block-so-that-the-signer-can-never-g.md)
* [38398 - [BC - High] Malicious Signers can initiate repeated contract calls to cause the multi-sign wallet to lose tx fee](./38398-bc-high-malicious-signers-can-initiate-repeated-contract-calls-to-cause-the-multi-sign-wallet.md)
* [37479 - [BC - High] A single signer can lock users' funds by not notifying other signers of the executed \`sweep\` transaction](./37479-bc-high-a-single-signer-can-lock-users-funds-by-not-notifying-other-signers-of-the-executed-sw.md)
* [38516 - [BC - High] Signer can censor transactions and halt the network by providing an invalid nonce or too many nonces](./38516-bc-high-signer-can-censor-transactions-and-halt-the-network-by-providing-an-invalid-nonce-or-t.md)

</details>

<details>

<summary>Medium</summary>

* [37777 - [BC - Medium] \`Emily.create\_deposit\` can overwrite any deposit to the Pending state](./37777-bc-medium-emily.create_deposit-can-overwrite-any-deposit-to-the-pending-state.md)
* [38133 - [BC - Medium] A rogue Signer can censor any deposit request from being processed and fullfilled on the Stacks blockchain](./38133-bc-medium-a-rogue-signer-can-censor-any-deposit-request-from-being-processed-and-fullfilled-on.md)
* [37384 - [BC - Medium] Attacker can front-run call to emily api with incorrect data, preventing legit user from registering their deposit](./37384-bc-medium-attacker-can-front-run-call-to-emily-api-with-incorrect-data-preventing-legit-user-f.md)
* [38551 - [BC - Medium] A signer can request stacks tx nonces in batches in advance and then DoS other signers' sBTC contract calls](./38551-bc-medium-a-signer-can-request-stacks-tx-nonces-in-batches-in-advance-and-then-dos-other-signe.md)
* [37470 - [BC - Medium] SBTC Signers do not page through pending deposit requests making it trivially easy to block legit deposits by spamming Emily API](./37470-bc-medium-sbtc-signers-do-not-page-through-pending-deposit-requests-making-it-trivially-easy-t.md)
* [38270 - [BC - Medium] A signer can send a large number of junk \`WstsNetMessage::NonceRequest\` through P2P to make other signers run out of memory](./38270-bc-medium-a-signer-can-send-a-large-number-of-junk-wstsnetmessage-noncerequest-through-p2p-to.md)
* [38003 - [BC - Medium] A malicious coordinator calling \`Emily::update\_deposits\` can make the entire Signers network inoperable](./38003-bc-medium-a-malicious-coordinator-calling-emily-update_deposits-can-make-the-entire-signers-ne.md)
* [37545 - [BC - Medium] Deposits with a lock\_time of 16 cannot be processed](./37545-bc-medium-deposits-with-a-lock_time-of-16-cannot-be-processed.md)

</details>

<details>

<summary>Low</summary>

* [38605 - [BC - Low] Lack of fee\_rate/last\_fees validation in handle\_bitcoin\_pre\_sign\_request ebables rogue signer to cause financial loss to depositors](./38605-bc-low-lack-of-fee_rate-last_fees-validation-in-handle_bitcoin_pre_sign_request-ebables-rogue.md)
* [38028 - [BC - Low] There is a Partial Network Degradation Due to DynamoDB GSI Throttling Under High Traffic](./38028-bc-low-there-is-a-partial-network-degradation-due-to-dynamodb-gsi-throttling-under-high-traffi.md)
* [38460 - [BC - Low] The coordinator can set a higher BTC tx fee than the current network to make users to pay more fees to the BTC miner](./38460-bc-low-the-coordinator-can-set-a-higher-btc-tx-fee-than-the-current-network-to-make-users-to-p.md)
* [37500 - [BC - Low] Blocklist can be circumvented due to incorrect blocking logic in \`request\_decider::can\_accept\_deposit\_request\`](./37500-bc-low-blocklist-can-be-circumvented-due-to-incorrect-blocking-logic-in-request_decider-can_ac.md)

</details>

<details>

<summary>Insight</summary>

* [38671 - [BC - Insight] Signer key rotation is not possible due to deadlock between submitting key rotation to Stacks and retrieving it](./38671-bc-insight-signer-key-rotation-is-not-possible-due-to-deadlock-between-submitting-key-rotation.md)
* [38030 - [BC - Insight] Coordinator can be crashed by signers on DKG](./38030-bc-insight-coordinator-can-be-crashed-by-signers-on-dkg.md)
* [38223 - [BC - Insight] Attackers can disrupt the tag order of gossip messages to bypass signature verification](./38223-bc-insight-attackers-can-disrupt-the-tag-order-of-gossip-messages-to-bypass-signature-verifica.md)
* [38690 - [BC - Insight] A malicious coordinator can run multiple DKG coordination in parallel and manipulate their order to break the signers network](./38690-bc-insight-a-malicious-coordinator-can-run-multiple-dkg-coordination-in-parallel-and-manipulat.md)
* [38160 - [BC - Insight] Governance calling \`sbtc-registry.update-protocol-contract\` may cause Stacks' events to be ignored by the signer](./38160-bc-insight-governance-calling-sbtc-registry.update-protocol-contract-may-cause-stacks-events-t.md)
* [37530 - [BC - Insight] Deposits can be completely DoSed due to incorrect transaction construction](./37530-bc-insight-deposits-can-be-completely-dosed-due-to-incorrect-transaction-construction.md)

</details>

## Reports by Type

[Blockchain/DLT](<README.md#blockchain-dlt>)
<details>

<summary>Blockchain/DLT</summary>

* [37718 - [BC - High] Key rotations bricks the system due to incorrect \`aggregate\_key\` being used to spend the \`peg UTXO\` when signing a sweep transaction](./37718-bc-high-key-rotations-bricks-the-system-due-to-incorrect-aggregate_key-being-used-to-spend-the.md)
* [37777 - [BC - Medium] \`Emily.create\_deposit\` can overwrite any deposit to the Pending state](./37777-bc-medium-emily.create_deposit-can-overwrite-any-deposit-to-the-pending-state.md)
* [37811 - [BC - High] Missing length check when parsing \`SignatureShareRequest\` in the signers allows the coordinator to halt other signers, shutting down the network](./37811-bc-high-missing-length-check-when-parsing-signaturesharerequest-in-the-signers-allows-the-coor.md)
* [37814 - [BC - High] Signers can crash other signers by sending an invalid \`DkgPrivateShares\` due to missing check before passing the payload to \`SignerStateMachine::process\`](./37814-bc-high-signers-can-crash-other-signers-by-sending-an-invalid-dkgprivateshares-due-to-missing.md)
* [38582 - [BC - High] The \`BitcoinCoreClient::get\_tx\_info\` does not support coinbase transactions, which may cause sBTC to be attacked by btc miners or sBTC donations to be lost](./38582-bc-high-the-bitcoincoreclient-get_tx_info-does-not-support-coinbase-transactions-which-may-cau.md)
* [38605 - [BC - Low] Lack of fee\_rate/last\_fees validation in handle\_bitcoin\_pre\_sign\_request ebables rogue signer to cause financial loss to depositors](./38605-bc-low-lack-of-fee_rate-last_fees-validation-in-handle_bitcoin_pre_sign_request-ebables-rogue.md)
* [37861 - [BC - Critical] SBTC Signer WSTS implementation allows nonce replays such that a malicious signer can steal all funds](./37861-bc-critical-sbtc-signer-wsts-implementation-allows-nonce-replays-such-that-a-malicious-signer.md)
* [38392 - [BC - High] Signer can steal STX tokens in multi-sign wallet by setting a high stacks tx fee](./38392-bc-high-signer-can-steal-stx-tokens-in-multi-sign-wallet-by-setting-a-high-stacks-tx-fee.md)
* [38671 - [BC - Insight] Signer key rotation is not possible due to deadlock between submitting key rotation to Stacks and retrieving it](./38671-bc-insight-signer-key-rotation-is-not-possible-due-to-deadlock-between-submitting-key-rotation.md)
* [38458 - [BC - Critical] The coordinator can submit empty BTC transactions to drain BTC tokens in the multi-sign wallet](./38458-bc-critical-the-coordinator-can-submit-empty-btc-transactions-to-drain-btc-tokens-in-the-multi.md)
* [38028 - [BC - Low] There is a Partial Network Degradation Due to DynamoDB GSI Throttling Under High Traffic](./38028-bc-low-there-is-a-partial-network-degradation-due-to-dynamodb-gsi-throttling-under-high-traffi.md)
* [38030 - [BC - Insight] Coordinator can be crashed by signers on DKG](./38030-bc-insight-coordinator-can-be-crashed-by-signers-on-dkg.md)
* [38740 - [BC - High] The missing check in Deposits::DepositScriptInputs::parse() permits losing funds by sending them to an invalid principal](./38740-bc-high-the-missing-check-in-deposits-depositscriptinputs-parse-permits-losing-funds-by-sendin.md)
* [38053 - [BC - High] A single signer can continuously prevent signatures from being finalized, halting network operations](./38053-bc-high-a-single-signer-can-continuously-prevent-signatures-from-being-finalized-halting-netwo.md)
* [38133 - [BC - Medium] A rogue Signer can censor any deposit request from being processed and fullfilled on the Stacks blockchain](./38133-bc-medium-a-rogue-signer-can-censor-any-deposit-request-from-being-processed-and-fullfilled-on.md)
* [37384 - [BC - Medium] Attacker can front-run call to emily api with incorrect data, preventing legit user from registering their deposit](./37384-bc-medium-attacker-can-front-run-call-to-emily-api-with-incorrect-data-preventing-legit-user-f.md)
* [38460 - [BC - Low] The coordinator can set a higher BTC tx fee than the current network to make users to pay more fees to the BTC miner](./38460-bc-low-the-coordinator-can-set-a-higher-btc-tx-fee-than-the-current-network-to-make-users-to-p.md)
* [38477 - [BC - High] A single signer can abort every attempted signing round by providing an invalid packet once the coordinator requests signature shares](./38477-bc-high-a-single-signer-can-abort-every-attempted-signing-round-by-providing-an-invalid-packet.md)
* [38111 - [BC - High] Attackers can send a very large event in a Stacks block so that the Signer can never get the Stacks event](./38111-bc-high-attackers-can-send-a-very-large-event-in-a-stacks-block-so-that-the-signer-can-never-g.md)
* [38551 - [BC - Medium] A signer can request stacks tx nonces in batches in advance and then DoS other signers' sBTC contract calls](./38551-bc-medium-a-signer-can-request-stacks-tx-nonces-in-batches-in-advance-and-then-dos-other-signe.md)
* [37470 - [BC - Medium] SBTC Signers do not page through pending deposit requests making it trivially easy to block legit deposits by spamming Emily API](./37470-bc-medium-sbtc-signers-do-not-page-through-pending-deposit-requests-making-it-trivially-easy-t.md)
* [38223 - [BC - Insight] Attackers can disrupt the tag order of gossip messages to bypass signature verification](./38223-bc-insight-attackers-can-disrupt-the-tag-order-of-gossip-messages-to-bypass-signature-verifica.md)
* [38270 - [BC - Medium] A signer can send a large number of junk \`WstsNetMessage::NonceRequest\` through P2P to make other signers run out of memory](./38270-bc-medium-a-signer-can-send-a-large-number-of-junk-wstsnetmessage-noncerequest-through-p2p-to.md)
* [38690 - [BC - Insight] A malicious coordinator can run multiple DKG coordination in parallel and manipulate their order to break the signers network](./38690-bc-insight-a-malicious-coordinator-can-run-multiple-dkg-coordination-in-parallel-and-manipulat.md)
* [37500 - [BC - Low] Blocklist can be circumvented due to incorrect blocking logic in \`request\_decider::can\_accept\_deposit\_request\`](./37500-bc-low-blocklist-can-be-circumvented-due-to-incorrect-blocking-logic-in-request_decider-can_ac.md)
* [38160 - [BC - Insight] Governance calling \`sbtc-registry.update-protocol-contract\` may cause Stacks' events to be ignored by the signer](./38160-bc-insight-governance-calling-sbtc-registry.update-protocol-contract-may-cause-stacks-events-t.md)
* [37530 - [BC - Insight] Deposits can be completely DoSed due to incorrect transaction construction](./37530-bc-insight-deposits-can-be-completely-dosed-due-to-incorrect-transaction-construction.md)
* [38398 - [BC - High] Malicious Signers can initiate repeated contract calls to cause the multi-sign wallet to lose tx fee](./38398-bc-high-malicious-signers-can-initiate-repeated-contract-calls-to-cause-the-multi-sign-wallet.md)
* [37479 - [BC - High] A single signer can lock users' funds by not notifying other signers of the executed \`sweep\` transaction](./37479-bc-high-a-single-signer-can-lock-users-funds-by-not-notifying-other-signers-of-the-executed-sw.md)
* [38003 - [BC - Medium] A malicious coordinator calling \`Emily::update\_deposits\` can make the entire Signers network inoperable](./38003-bc-medium-a-malicious-coordinator-calling-emily-update_deposits-can-make-the-entire-signers-ne.md)
* [37545 - [BC - Medium] Deposits with a lock\_time of 16 cannot be processed](./37545-bc-medium-deposits-with-a-lock_time-of-16-cannot-be-processed.md)
* [38516 - [BC - High] Signer can censor transactions and halt the network by providing an invalid nonce or too many nonces](./38516-bc-high-signer-can-censor-transactions-and-halt-the-network-by-providing-an-invalid-nonce-or-t.md)

</details>
