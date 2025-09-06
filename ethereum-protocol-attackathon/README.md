---
hidden: true
---

# Ethereum Protocol Attackathon

## Reports by Severity

[Medium](<README.md#medium>) | [Low](<README.md#low>) | [Insight](<README.md#insight>)
<details>

<summary>Medium</summary>

* [37466 - [BC - Medium] Evil-client OOM crash (fast P2P crash)](./37466-bc-medium-evil-client-oom-crash-fast-p2p-crash.md)
* [38292 - [SC - Medium] Incorrect Sqrt Calculation Result](./38292-sc-medium-incorrect-sqrt-calculation-result.md)
* [38682 - [SC - Medium] AugAssign evaluation order causing OOB write within the object](./38682-sc-medium-augassign-evaluation-order-causing-oob-write-within-the-object.md)
* [38733 - [BC - Medium] nibmus-eth2 remote crash](./38733-bc-medium-nibmus-eth2-remote-crash.md)
* [38920 - [BC - Medium] teku remote DoS](./38920-bc-medium-teku-remote-dos.md)
* [38146 - [BC - Medium] nimbus-eth2 remote crash](./38146-bc-medium-nimbus-eth2-remote-crash.md)

</details>

<details>

<summary>Low</summary>

* [38502 - [BC - Low] Pending pool subtraction overflow causes node halt/shutdown](./38502-bc-low-pending-pool-subtraction-overflow-causes-node-halt-shutdown.md)
* [38505 - [SC - Low] IRNode Multi-Evaluation In For List Iter](./38505-sc-low-irnode-multi-evaluation-in-for-list-iter.md)
* [38530 - [SC - Low] Incorrectly Eliminated Code With Side Effect In Concat Args](./38530-sc-low-incorrectly-eliminated-code-with-side-effect-in-concat-args.md)
* [38554 - [BC - Low] Incorrect Transaction Fee Check in \`SendRawTransaction()\`](./38554-bc-low-incorrect-transaction-fee-check-in-sendrawtransaction.md)
* [37199 - [BC - Low] Potential Chain Fork Due to Shallow Copy of Byte Slice](./37199-bc-low-potential-chain-fork-due-to-shallow-copy-of-byte-slice.md)
* [37245 - [BC - Low] lodestar snappy decompression issue](./37245-bc-low-lodestar-snappy-decompression-issue.md)
* [38686 - [BC - Low] Nodes with trusted peers vulnerable to pending peer flooding and DoS](./38686-bc-low-nodes-with-trusted-peers-vulnerable-to-pending-peer-flooding-and-dos.md)
* [37985 - [SC - Low] Incorrectly Eliminate Code With Side Effect In Slice Args](./37985-sc-low-incorrectly-eliminate-code-with-side-effect-in-slice-args.md)
* [38807 - [BC - Low] DoS any reth node via ban logic exploit](./38807-bc-low-dos-any-reth-node-via-ban-logic-exploit.md)
* [37462 - [BC - Low] Invalid RLP decoding for single bytes](./37462-bc-low-invalid-rlp-decoding-for-single-bytes.md)
* [38850 - [BC - Low] Remote P2P OOM Crash (GetBlockHeaders) / Reth](./38850-bc-low-remote-p2p-oom-crash-getblockheaders-reth.md)
* [38855 - [SC - Low] Evaluation order is not respected in \`log\` function](./38855-sc-low-evaluation-order-is-not-respected-in-log-function.md)
* [38894 - [BC - Low] Missing expiration check for Pong and Neighbors packets and not refreshing the endpoint proof](./38894-bc-low-missing-expiration-check-for-pong-and-neighbors-packets-and-not-refreshing-the-endpoint.md)
* [38275 - [BC - Low] Evil-client P2P headers-traversal leads to D/DoS and total peer removal](./38275-bc-low-evil-client-p2p-headers-traversal-leads-to-d-dos-and-total-peer-removal.md)
* [37582 - [SC - Low] Incorrect HexString Parsing Leads To Compilation Error Or Type Confusion](./37582-sc-low-incorrect-hexstring-parsing-leads-to-compilation-error-or-type-confusion.md)
* [37583 - [SC - Low] Incorrect For Annotation Parsing](./37583-sc-low-incorrect-for-annotation-parsing.md)
* [38958 - [BC - Low] EELS cant handle overflow gas calculation in modexp precompile](./38958-bc-low-eels-cant-handle-overflow-gas-calculation-in-modexp-precompile.md)
* [38828 - [BC - Low] Decode RLP of Legacy Transaction Allows Tailing Bytes](./38828-bc-low-decode-rlp-of-legacy-transaction-allows-tailing-bytes.md)
<!-- * [37113 - [BC - Low] A potential out-of-range panic has been discovered in the Ethereum client Erigon ( https://github.com/erigontech/erigon ), though it does not seem to be exploitable at this moment du...]() -->
<!-- * [38459 - [BC - Low] erigon remote DoS]() -->
* [37246 - [BC - Low] lodestar snappy checksum issue](./37246-bc-low-lodestar-snappy-checksum-issue.md)
* [37634 - [SC - Low] Incorrect Builtin ERC4626 Call Signature](./37634-sc-low-incorrect-builtin-erc4626-call-signature.md)
* [38427 - [BC - Low] Discrepancy in Intrinsic Gas Calculation between Txpool and EVM Execution](./38427-bc-low-discrepancy-in-intrinsic-gas-calculation-between-txpool-and-evm-execution.md)
* [38902 - [BC - Low] No check on the maximum size of the encoded ENR on ENR\_RESPONSE packet](./38902-bc-low-no-check-on-the-maximum-size-of-the-encoded-enr-on-enr_response-packet.md)
* [38948 - [BC - Low] lighthouse remote DoS](./38948-bc-low-lighthouse-remote-dos.md)
* [38278 - [BC - Low] Potential DoS to Mempool Due to Missing Gas Limit Check](./38278-bc-low-potential-dos-to-mempool-due-to-missing-gas-limit-check.md)
* [38318 - [BC - Low] nimbus-eth2: Gossipsub misconfiguration allows malicious peers gossip malformed data without penalization](./38318-bc-low-nimbus-eth2-gossipsub-misconfiguration-allows-malicious-peers-gossip-malformed-data-wit.md)

</details>

<details>

<summary>Insight</summary>

* [37646 - [BC - Insight] No implementation of BLOB\_SIDECAR\_SUBNET\_COUNT with no issue and no PR in the GitHub](./37646-bc-insight-no-implementation-of-blob_sidecar_subnet_count-with-no-issue-and-no-pr-in-the-githu.md)
* [37134 - [BC - Insight] Improper secp256k sanitization](./37134-bc-insight-improper-secp256k-sanitization.md)
* [37695 - [BC - Insight] Executing transaction that has a wrong nonce might triggered a chain split due to mismatch stateroot](./37695-bc-insight-executing-transaction-that-has-a-wrong-nonce-might-triggered-a-chain-split-due-to-m.md)
* [37120 - [BC - Insight] Remote handshake-based TCP/30303 flooding leads to an out-of-memory crash](./37120-bc-insight-remote-handshake-based-tcp-30303-flooding-leads-to-an-out-of-memory-crash.md)
* [37191 - [BC - Insight] Unvalidated Field Names in Tuple ABI Parsing Causes Runtime Panic via reflect.StructOf](./37191-bc-insight-unvalidated-field-names-in-tuple-abi-parsing-causes-runtime-panic-via-reflect.struc.md)
* [37593 - [BC - Insight] Inconsistent Address Collision Check Against Precompile Contracts During Contract Deployment](./37593-bc-insight-inconsistent-address-collision-check-against-precompile-contracts-during-contract-d.md)
* [38598 - [BC - Insight] GetReceiptsMsg abuse leads to the DoS and/or crash of every EL client in the Ethereum network](./38598-bc-insight-getreceiptsmsg-abuse-leads-to-the-dos-and-or-crash-of-every-el-client-in-the-ethere.md)
* [37352 - [BC - Insight] Missing Liveness Check in \`collectTableNodes()\`](./37352-bc-insight-missing-liveness-check-in-collecttablenodes.md)
* [37359 - [BC - Insight] Failure to Generate ABI Binding in Golang](./37359-bc-insight-failure-to-generate-abi-binding-in-golang.md)
* [37351 - [BC - Insight] Resubscribe Deadlocks When Unsubscribing Within An Unblock Channel](./37351-bc-insight-resubscribe-deadlocks-when-unsubscribing-within-an-unblock-channel.md)
* [38766 - [BC - Insight] Nil Pointer Dereference Panics in encodePayload() of Blob Tx’s Encoding](./38766-bc-insight-nil-pointer-dereference-panics-in-encodepayload-of-blob-txs-encoding.md)
* [37442 - [BC - Insight] Potential Address Collision with Precompile Contract During Contract Deployment](./37442-bc-insight-potential-address-collision-with-precompile-contract-during-contract-deployment.md)
* [38169 - [SC - Insight] Deferred Evaluation Of \`Default\_Return\_Value\` May Skip Side Effect Execution](./38169-sc-insight-deferred-evaluation-of-default_return_value-may-skip-side-effect-execution.md)
* [37483 - [BC - Insight] There is a trace discrepancy for Nethermind when handling EOF from PUSH opcode](./37483-bc-insight-there-is-a-trace-discrepancy-for-nethermind-when-handling-eof-from-push-opcode.md)
* [37505 - [BC - Insight] Remotely spamming 1 byte leads to full peer removal and desync in both execution and consensus clients](./37505-bc-insight-remotely-spamming-1-byte-leads-to-full-peer-removal-and-desync-in-both-execution-an.md)
* [37568 - [BC - Insight] Missing Specification Logic](./37568-bc-insight-missing-specification-logic.md)
* [37300 - [BC - Insight] Incorrect Encoding of Negative \*big.Int Values in MakeTopics](./37300-bc-insight-incorrect-encoding-of-negative-big.int-values-in-maketopics.md)
* [38277 - [BC - Insight] Potential Out-of-Range Panic in \`UnmarshalJSON()\` of \`HexOrDecimal256\`](./38277-bc-insight-potential-out-of-range-panic-in-unmarshaljson-of-hexordecimal256.md)
* [38908 - [BC - Insight] Missing Failed Subcalls in Erigon Tracers When Encountering \`ErrInsufficientBalance\` Error](./38908-bc-insight-missing-failed-subcalls-in-erigon-tracers-when-encountering-errinsufficientbalance.md)
* [39018 - [BC - Insight] Rate Limiting Under-Specification and Consequences](./39018-bc-insight-rate-limiting-under-specification-and-consequences.md)
* [37286 - [SC - Insight] Elimination of Security Checks in ForkCreator Class](./37286-sc-insight-elimination-of-security-checks-in-forkcreator-class.md)
* [37148 - [BC - Insight] \`wantedPeerDials()\` branch will never be executed](./37148-bc-insight-wantedpeerdials-branch-will-never-be-executed.md)
* [38557 - [BC - Insight] Function \`IsPush()\` Misses Opcode PUSH0](./38557-bc-insight-function-ispush-misses-opcode-push0.md)
* [37584 - [SC - Insight] Nonpayable Not Respected For Internal Function](./37584-sc-insight-nonpayable-not-respected-for-internal-function.md)
* [38581 - [SC - Insight] Incorrect unwrap on Bytes and String](./38581-sc-insight-incorrect-unwrap-on-bytes-and-string.md)
* [37210 - [BC - Insight] Missing Check of HTTP Batch Response Length](./37210-bc-insight-missing-check-of-http-batch-response-length.md)
* [37350 - [BC - Insight] \`null\` Is Not Unmarshalled Correctly Into json.RawMessage](./37350-bc-insight-null-is-not-unmarshalled-correctly-into-json.rawmessage.md)
* [37104 - [BC - Insight] Reth RPC is vulnerable to DNS rebinding attacks](./37104-bc-insight-reth-rpc-is-vulnerable-to-dns-rebinding-attacks.md)
* [38015 - [BC - Insight] Violation of EIP-2681 in Create Transaction](./38015-bc-insight-violation-of-eip-2681-in-create-transaction.md)
* [37186 - [BC - Insight] Missing Validation for Fixed-Size bytes Types in ABI Parsing](./37186-bc-insight-missing-validation-for-fixed-size-bytes-types-in-abi-parsing.md)
* [38319 - [BC - Insight] Edge case difference for GETH and NETHERMIND when calculating memory expansion gas](./38319-bc-insight-edge-case-difference-for-geth-and-nethermind-when-calculating-memory-expansion-gas.md)
* [37594 - [SC - Insight] Nimbus incorrectly rejects non-minimally encoded snappy data length's due to spec. ambiguity](./37594-sc-insight-nimbus-incorrectly-rejects-non-minimally-encoded-snappy-data-lengths-due-to-spec.-a.md)
* [37153 - [BC - Insight] Malicious validator can bring down honest nodes](./37153-bc-insight-malicious-validator-can-bring-down-honest-nodes.md)
* [37577 - [BC - Insight] \`tx.origin\` Usage in Group Management Contract Allows Phishing Attack for Unauthorized Actions](./37577-bc-insight-tx.origin-usage-in-group-management-contract-allows-phishing-attack-for-unauthorize.md)
* [38693 - [SC - Insight] BytesM to Bytes conversion does not match the reference implementation](./38693-sc-insight-bytesm-to-bytes-conversion-does-not-match-the-reference-implementation.md)

</details>

## Reports by Type

[Smart Contract](<README.md#smart-contract>)
<details>

<summary>Smart Contract</summary>

* [38505 - [SC - Low] IRNode Multi-Evaluation In For List Iter](./38505-sc-low-irnode-multi-evaluation-in-for-list-iter.md)
* [38530 - [SC - Low] Incorrectly Eliminated Code With Side Effect In Concat Args](./38530-sc-low-incorrectly-eliminated-code-with-side-effect-in-concat-args.md)
* [37985 - [SC - Low] Incorrectly Eliminate Code With Side Effect In Slice Args](./37985-sc-low-incorrectly-eliminate-code-with-side-effect-in-slice-args.md)
* [38169 - [SC - Insight] Deferred Evaluation Of \`Default\_Return\_Value\` May Skip Side Effect Execution](./38169-sc-insight-deferred-evaluation-of-default_return_value-may-skip-side-effect-execution.md)
* [38855 - [SC - Low] Evaluation order is not respected in \`log\` function](./38855-sc-low-evaluation-order-is-not-respected-in-log-function.md)
* [37582 - [SC - Low] Incorrect HexString Parsing Leads To Compilation Error Or Type Confusion](./37582-sc-low-incorrect-hexstring-parsing-leads-to-compilation-error-or-type-confusion.md)
* [37583 - [SC - Low] Incorrect For Annotation Parsing](./37583-sc-low-incorrect-for-annotation-parsing.md)
* [38292 - [SC - Medium] Incorrect Sqrt Calculation Result](./38292-sc-medium-incorrect-sqrt-calculation-result.md)
* [38682 - [SC - Medium] AugAssign evaluation order causing OOB write within the object](./38682-sc-medium-augassign-evaluation-order-causing-oob-write-within-the-object.md)
* [37286 - [SC - Insight] Elimination of Security Checks in ForkCreator Class](./37286-sc-insight-elimination-of-security-checks-in-forkcreator-class.md)
* [37634 - [SC - Low] Incorrect Builtin ERC4626 Call Signature](./37634-sc-low-incorrect-builtin-erc4626-call-signature.md)
* [37584 - [SC - Insight] Nonpayable Not Respected For Internal Function](./37584-sc-insight-nonpayable-not-respected-for-internal-function.md)
* [38581 - [SC - Insight] Incorrect unwrap on Bytes and String](./38581-sc-insight-incorrect-unwrap-on-bytes-and-string.md)
* [37594 - [SC - Insight] Nimbus incorrectly rejects non-minimally encoded snappy data length's due to spec. ambiguity](./37594-sc-insight-nimbus-incorrectly-rejects-non-minimally-encoded-snappy-data-lengths-due-to-spec.-a.md)
* [38693 - [SC - Insight] BytesM to Bytes conversion does not match the reference implementation](./38693-sc-insight-bytesm-to-bytes-conversion-does-not-match-the-reference-implementation.md)

</details>

[Blockchain/DLT](<README.md#blockchain-dlt>)
<details>

<summary>Blockchain/DLT</summary>

* [37646 - [BC - Insight] No implementation of BLOB\_SIDECAR\_SUBNET\_COUNT with no issue and no PR in the GitHub](./37646-bc-insight-no-implementation-of-blob_sidecar_subnet_count-with-no-issue-and-no-pr-in-the-githu.md)
* [38502 - [BC - Low] Pending pool subtraction overflow causes node halt/shutdown](./38502-bc-low-pending-pool-subtraction-overflow-causes-node-halt-shutdown.md)
* [38554 - [BC - Low] Incorrect Transaction Fee Check in \`SendRawTransaction()\`](./38554-bc-low-incorrect-transaction-fee-check-in-sendrawtransaction.md)
* [37134 - [BC - Insight] Improper secp256k sanitization](./37134-bc-insight-improper-secp256k-sanitization.md)
* [37695 - [BC - Insight] Executing transaction that has a wrong nonce might triggered a chain split due to mismatch stateroot](./37695-bc-insight-executing-transaction-that-has-a-wrong-nonce-might-triggered-a-chain-split-due-to-m.md)
* [37120 - [BC - Insight] Remote handshake-based TCP/30303 flooding leads to an out-of-memory crash](./37120-bc-insight-remote-handshake-based-tcp-30303-flooding-leads-to-an-out-of-memory-crash.md)
* [37191 - [BC - Insight] Unvalidated Field Names in Tuple ABI Parsing Causes Runtime Panic via reflect.StructOf](./37191-bc-insight-unvalidated-field-names-in-tuple-abi-parsing-causes-runtime-panic-via-reflect.struc.md)
* [37199 - [BC - Low] Potential Chain Fork Due to Shallow Copy of Byte Slice](./37199-bc-low-potential-chain-fork-due-to-shallow-copy-of-byte-slice.md)
* [37245 - [BC - Low] lodestar snappy decompression issue](./37245-bc-low-lodestar-snappy-decompression-issue.md)
* [38686 - [BC - Low] Nodes with trusted peers vulnerable to pending peer flooding and DoS](./38686-bc-low-nodes-with-trusted-peers-vulnerable-to-pending-peer-flooding-and-dos.md)
* [37593 - [BC - Insight] Inconsistent Address Collision Check Against Precompile Contracts During Contract Deployment](./37593-bc-insight-inconsistent-address-collision-check-against-precompile-contracts-during-contract-d.md)
* [38598 - [BC - Insight] GetReceiptsMsg abuse leads to the DoS and/or crash of every EL client in the Ethereum network](./38598-bc-insight-getreceiptsmsg-abuse-leads-to-the-dos-and-or-crash-of-every-el-client-in-the-ethere.md)
* [37466 - [BC - Medium] Evil-client OOM crash (fast P2P crash)](./37466-bc-medium-evil-client-oom-crash-fast-p2p-crash.md)
* [37352 - [BC - Insight] Missing Liveness Check in \`collectTableNodes()\`](./37352-bc-insight-missing-liveness-check-in-collecttablenodes.md)
* [37359 - [BC - Insight] Failure to Generate ABI Binding in Golang](./37359-bc-insight-failure-to-generate-abi-binding-in-golang.md)
* [37351 - [BC - Insight] Resubscribe Deadlocks When Unsubscribing Within An Unblock Channel](./37351-bc-insight-resubscribe-deadlocks-when-unsubscribing-within-an-unblock-channel.md)
* [38766 - [BC - Insight] Nil Pointer Dereference Panics in encodePayload() of Blob Tx’s Encoding](./38766-bc-insight-nil-pointer-dereference-panics-in-encodepayload-of-blob-txs-encoding.md)
* [38807 - [BC - Low] DoS any reth node via ban logic exploit](./38807-bc-low-dos-any-reth-node-via-ban-logic-exploit.md)
* [37442 - [BC - Insight] Potential Address Collision with Precompile Contract During Contract Deployment](./37442-bc-insight-potential-address-collision-with-precompile-contract-during-contract-deployment.md)
* [37462 - [BC - Low] Invalid RLP decoding for single bytes](./37462-bc-low-invalid-rlp-decoding-for-single-bytes.md)
* [37483 - [BC - Insight] There is a trace discrepancy for Nethermind when handling EOF from PUSH opcode](./37483-bc-insight-there-is-a-trace-discrepancy-for-nethermind-when-handling-eof-from-push-opcode.md)
* [38850 - [BC - Low] Remote P2P OOM Crash (GetBlockHeaders) / Reth](./38850-bc-low-remote-p2p-oom-crash-getblockheaders-reth.md)
* [37505 - [BC - Insight] Remotely spamming 1 byte leads to full peer removal and desync in both execution and consensus clients](./37505-bc-insight-remotely-spamming-1-byte-leads-to-full-peer-removal-and-desync-in-both-execution-an.md)
* [37568 - [BC - Insight] Missing Specification Logic](./37568-bc-insight-missing-specification-logic.md)
* [38894 - [BC - Low] Missing expiration check for Pong and Neighbors packets and not refreshing the endpoint proof](./38894-bc-low-missing-expiration-check-for-pong-and-neighbors-packets-and-not-refreshing-the-endpoint.md)
* [38275 - [BC - Low] Evil-client P2P headers-traversal leads to D/DoS and total peer removal](./38275-bc-low-evil-client-p2p-headers-traversal-leads-to-d-dos-and-total-peer-removal.md)
* [37300 - [BC - Insight] Incorrect Encoding of Negative \*big.Int Values in MakeTopics](./37300-bc-insight-incorrect-encoding-of-negative-big.int-values-in-maketopics.md)
* [38277 - [BC - Insight] Potential Out-of-Range Panic in \`UnmarshalJSON()\` of \`HexOrDecimal256\`](./38277-bc-insight-potential-out-of-range-panic-in-unmarshaljson-of-hexordecimal256.md)
* [38908 - [BC - Insight] Missing Failed Subcalls in Erigon Tracers When Encountering \`ErrInsufficientBalance\` Error](./38908-bc-insight-missing-failed-subcalls-in-erigon-tracers-when-encountering-errinsufficientbalance.md)
* [38958 - [BC - Low] EELS cant handle overflow gas calculation in modexp precompile](./38958-bc-low-eels-cant-handle-overflow-gas-calculation-in-modexp-precompile.md)
* [38828 - [BC - Low] Decode RLP of Legacy Transaction Allows Tailing Bytes](./38828-bc-low-decode-rlp-of-legacy-transaction-allows-tailing-bytes.md)
* [39018 - [BC - Insight] Rate Limiting Under-Specification and Consequences](./39018-bc-insight-rate-limiting-under-specification-and-consequences.md)
<!-- * [37113 - [BC - Low] A potential out-of-range panic has been discovered in the Ethereum client Erigon ( https://github.com/erigontech/erigon ), though it does not seem to be exploitable at this moment du...]() -->
<!-- * [38459 - [BC - Low] erigon remote DoS]() -->
* [37246 - [BC - Low] lodestar snappy checksum issue](./37246-bc-low-lodestar-snappy-checksum-issue.md)
* [38733 - [BC - Medium] nibmus-eth2 remote crash](./38733-bc-medium-nibmus-eth2-remote-crash.md)
* [38920 - [BC - Medium] teku remote DoS](./38920-bc-medium-teku-remote-dos.md)
* [37148 - [BC - Insight] \`wantedPeerDials()\` branch will never be executed](./37148-bc-insight-wantedpeerdials-branch-will-never-be-executed.md)
* [38557 - [BC - Insight] Function \`IsPush()\` Misses Opcode PUSH0](./38557-bc-insight-function-ispush-misses-opcode-push0.md)
* [38427 - [BC - Low] Discrepancy in Intrinsic Gas Calculation between Txpool and EVM Execution](./38427-bc-low-discrepancy-in-intrinsic-gas-calculation-between-txpool-and-evm-execution.md)
* [37210 - [BC - Insight] Missing Check of HTTP Batch Response Length](./37210-bc-insight-missing-check-of-http-batch-response-length.md)
* [38902 - [BC - Low] No check on the maximum size of the encoded ENR on ENR\_RESPONSE packet](./38902-bc-low-no-check-on-the-maximum-size-of-the-encoded-enr-on-enr_response-packet.md)
* [37350 - [BC - Insight] \`null\` Is Not Unmarshalled Correctly Into json.RawMessage](./37350-bc-insight-null-is-not-unmarshalled-correctly-into-json.rawmessage.md)
* [37104 - [BC - Insight] Reth RPC is vulnerable to DNS rebinding attacks](./37104-bc-insight-reth-rpc-is-vulnerable-to-dns-rebinding-attacks.md)
* [38948 - [BC - Low] lighthouse remote DoS](./38948-bc-low-lighthouse-remote-dos.md)
* [38015 - [BC - Insight] Violation of EIP-2681 in Create Transaction](./38015-bc-insight-violation-of-eip-2681-in-create-transaction.md)
* [37186 - [BC - Insight] Missing Validation for Fixed-Size bytes Types in ABI Parsing](./37186-bc-insight-missing-validation-for-fixed-size-bytes-types-in-abi-parsing.md)
* [38319 - [BC - Insight] Edge case difference for GETH and NETHERMIND when calculating memory expansion gas](./38319-bc-insight-edge-case-difference-for-geth-and-nethermind-when-calculating-memory-expansion-gas.md)
* [37153 - [BC - Insight] Malicious validator can bring down honest nodes](./37153-bc-insight-malicious-validator-can-bring-down-honest-nodes.md)
* [38278 - [BC - Low] Potential DoS to Mempool Due to Missing Gas Limit Check](./38278-bc-low-potential-dos-to-mempool-due-to-missing-gas-limit-check.md)
* [38318 - [BC - Low] nimbus-eth2: Gossipsub misconfiguration allows malicious peers gossip malformed data without penalization](./38318-bc-low-nimbus-eth2-gossipsub-misconfiguration-allows-malicious-peers-gossip-malformed-data-wit.md)
* [37577 - [BC - Insight] \`tx.origin\` Usage in Group Management Contract Allows Phishing Attack for Unauthorized Actions](./37577-bc-insight-tx.origin-usage-in-group-management-contract-allows-phishing-attack-for-unauthorize.md)
* [38146 - [BC - Medium] nimbus-eth2 remote crash](./38146-bc-medium-nimbus-eth2-remote-crash.md)

</details>
