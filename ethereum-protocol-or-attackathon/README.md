# Ethereum Protocol | Attackathon

## Reports by Severity

<details>

<summary>Medium</summary>

* \#37466 \[BC-Medium] Evil-client OOM crash (fast P2P crash)
* \#38292 \[SC-Medium] Incorrect Sqrt Calculation Result
* \#38682 \[SC-Medium] AugAssign evaluation order causing OOB write within the object
* \#38733 \[BC-Medium] nibmus-eth2 remote crash
* \#38920 \[BC-Medium] teku remote DoS
* \#38146 \[BC-Medium] nimbus-eth2 remote crash

</details>

<details>

<summary>Low</summary>

* \#38502 \[BC-Low] Pending pool subtraction overflow causes node halt/shutdown
* \#38505 \[SC-Low] IRNode Multi-Evaluation In For List Iter
* \#38530 \[SC-Low] Incorrectly Eliminated Code With Side Effect In Concat Args
* \#38554 \[BC-Low] Incorrect Transaction Fee Check in \`SendRawTransaction()\`
* \#37199 \[BC-Low] Potential Chain Fork Due to Shallow Copy of Byte Slice
* \#38686 \[BC-Low] Nodes with trusted peers vulnerable to pending peer flooding and DoS
* \#37985 \[SC-Low] Incorrectly Eliminate Code With Side Effect In Slice Args
* \#38807 \[BC-Low] DoS any reth node via ban logic exploit
* \#37462 \[BC-Low] Invalid RLP decoding for single bytes
* \#38850 \[BC-Low] Remote P2P OOM Crash (GetBlockHeaders) / Reth
* \#38855 \[SC-Low] Evaluation order is not respected in \`log\` function
* \#38894 \[BC-Low] Missing expiration check for Pong and Neighbors packets and not refreshing the endpoint proof
* \#38275 \[BC-Low] Evil-client P2P headers-traversal leads to D/DoS and total peer removal
* \#37582 \[SC-Low] Incorrect HexString Parsing Leads To Compilation Error Or Type Confusion
* \#37583 \[SC-Low] Incorrect For Annotation Parsing
* \#38958 \[BC-Low] EELS cant handle overflow gas calculation in modexp precompile
* \#38828 \[BC-Low] Decode RLP of Legacy Transaction Allows Tailing Bytes
* \#37113 \[BC-Low] A potential out-of-range panic has been discovered in the Ethereum client Erigon ( <https://github.com/erigontech/erigon> ), though it does not seem to be exploitable at this moment du...
* \#38459 \[BC-Low] erigon remote DoS
* \#37246 \[BC-Low] lodestar snappy checksum issue
* \#37634 \[SC-Low] Incorrect Builtin ERC4626 Call Signature
* \#38427 \[BC-Low] Discrepancy in Intrinsic Gas Calculation between Txpool and EVM Execution
* \#38902 \[BC-Low] No check on the maximum size of the encoded ENR on ENR\_RESPONSE packet
* \#38948 \[BC-Low] lighthouse remote DoS
* \#38278 \[BC-Low] Potential DoS to Mempool Due to Missing Gas Limit Check
* \#38318 \[BC-Low] nimbus-eth2: Gossipsub misconfiguration allows malicious peers gossip malformed data without penalization

</details>

<details>

<summary>Insight</summary>

* \#37646 \[BC-Insight] No implementation of BLOB\_SIDECAR\_SUBNET\_COUNT with no issue and no PR in the GitHub
* \#37134 \[BC-Insight] Improper secp256k sanitization
* \#37695 \[BC-Insight] Executing transaction that has a wrong nonce might triggered a chain split due to mismatch stateroot
* \#37120 \[BC-Insight] Remote handshake-based TCP/30303 flooding leads to an out-of-memory crash
* \#37191 \[BC-Insight] Unvalidated Field Names in Tuple ABI Parsing Causes Runtime Panic via reflect.StructOf
* \#37593 \[BC-Insight] Inconsistent Address Collision Check Against Precompile Contracts During Contract Deployment
* \#38598 \[BC-Insight] GetReceiptsMsg abuse leads to the DoS and/or crash of every EL client in the Ethereum network
* \#37352 \[BC-Insight] Missing Liveness Check in \`collectTableNodes()\`
* \#37359 \[BC-Insight] Failure to Generate ABI Binding in Golang
* \#37351 \[BC-Insight] Resubscribe Deadlocks When Unsubscribing Within An Unblock Channel
* \#38766 \[BC-Insight] Nil Pointer Dereference Panics in encodePayload() of Blob Tx’s Encoding
* \#37442 \[BC-Insight] Potential Address Collision with Precompile Contract During Contract Deployment
* \#38169 \[SC-Insight] Deferred Evaluation Of \`Default\_Return\_Value\` May Skip Side Effect Execution
* \#37483 \[BC-Insight] There is a trace discrepancy for Nethermind when handling EOF from PUSH opcode
* \#37505 \[BC-Insight] Remotely spamming 1 byte leads to full peer removal and desync in both execution and consensus clients
* \#37568 \[BC-Insight] Missing Specification Logic
* \#37300 \[BC-Insight] Incorrect Encoding of Negative \*big.Int Values in MakeTopics
* \#38277 \[BC-Insight] Potential Out-of-Range Panic in \`UnmarshalJSON()\` of \`HexOrDecimal256\`
* \#38908 \[BC-Insight] Missing Failed Subcalls in Erigon Tracers When Encountering \`ErrInsufficientBalance\` Error
* \#39018 \[BC-Insight] Rate Limiting Under-Specification and Consequences
* \#37286 \[SC-Insight] Elimination of Security Checks in ForkCreator Class
* \#37148 \[BC-Insight] \`wantedPeerDials()\` branch will never be executed
* \#38557 \[BC-Insight] Function \`IsPush()\` Misses Opcode PUSH0
* \#37584 \[SC-Insight] Nonpayable Not Respected For Internal Function
* \#38581 \[SC-Insight] Incorrect unwrap on Bytes and String
* \#37210 \[BC-Insight] Missing Check of HTTP Batch Response Length
* \#37350 \[BC-Insight] \`null\` Is Not Unmarshalled Correctly Into json.RawMessage
* \#37104 \[BC-Insight] Reth RPC is vulnerable to DNS rebinding attacks
* \#38015 \[BC-Insight] Violation of EIP-2681 in Create Transaction
* \#37186 \[BC-Insight] Missing Validation for Fixed-Size bytes Types in ABI Parsing
* \#38319 \[BC-Insight] Edge case difference for GETH and NETHERMIND when calculating memory expansion gas
* \#37594 \[SC-Insight] Nimbus incorrectly rejects non-minimally encoded snappy data length's due to spec. ambiguity
* \#37153 \[BC-Insight] Malicious validator can bring down honest nodes
* \#37577 \[BC-Insight] \`tx.origin\` Usage in Group Management Contract Allows Phishing Attack for Unauthorized Actions
* \#38693 \[SC-Insight] BytesM to Bytes conversion does not match the reference implementation

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#38505 \[SC-Low] IRNode Multi-Evaluation In For List Iter
* \#38530 \[SC-Low] Incorrectly Eliminated Code With Side Effect In Concat Args
* \#37985 \[SC-Low] Incorrectly Eliminate Code With Side Effect In Slice Args
* \#38169 \[SC-Insight] Deferred Evaluation Of \`Default\_Return\_Value\` May Skip Side Effect Execution
* \#38855 \[SC-Low] Evaluation order is not respected in \`log\` function
* \#37582 \[SC-Low] Incorrect HexString Parsing Leads To Compilation Error Or Type Confusion
* \#37583 \[SC-Low] Incorrect For Annotation Parsing
* \#38292 \[SC-Medium] Incorrect Sqrt Calculation Result
* \#38682 \[SC-Medium] AugAssign evaluation order causing OOB write within the object
* \#37286 \[SC-Insight] Elimination of Security Checks in ForkCreator Class
* \#37634 \[SC-Low] Incorrect Builtin ERC4626 Call Signature
* \#37584 \[SC-Insight] Nonpayable Not Respected For Internal Function
* \#38581 \[SC-Insight] Incorrect unwrap on Bytes and String
* \#37594 \[SC-Insight] Nimbus incorrectly rejects non-minimally encoded snappy data length's due to spec. ambiguity
* \#38693 \[SC-Insight] BytesM to Bytes conversion does not match the reference implementation

</details>

<details>

<summary>Blockchain/DLT</summary>

* \#37646 \[BC-Insight] No implementation of BLOB\_SIDECAR\_SUBNET\_COUNT with no issue and no PR in the GitHub
* \#38502 \[BC-Low] Pending pool subtraction overflow causes node halt/shutdown
* \#38554 \[BC-Low] Incorrect Transaction Fee Check in \`SendRawTransaction()\`
* \#37134 \[BC-Insight] Improper secp256k sanitization
* \#37695 \[BC-Insight] Executing transaction that has a wrong nonce might triggered a chain split due to mismatch stateroot
* \#37120 \[BC-Insight] Remote handshake-based TCP/30303 flooding leads to an out-of-memory crash
* \#37191 \[BC-Insight] Unvalidated Field Names in Tuple ABI Parsing Causes Runtime Panic via reflect.StructOf
* \#37199 \[BC-Low] Potential Chain Fork Due to Shallow Copy of Byte Slice
* \#38686 \[BC-Low] Nodes with trusted peers vulnerable to pending peer flooding and DoS
* \#37593 \[BC-Insight] Inconsistent Address Collision Check Against Precompile Contracts During Contract Deployment
* \#38598 \[BC-Insight] GetReceiptsMsg abuse leads to the DoS and/or crash of every EL client in the Ethereum network
* \#37466 \[BC-Medium] Evil-client OOM crash (fast P2P crash)
* \#37352 \[BC-Insight] Missing Liveness Check in \`collectTableNodes()\`
* \#37359 \[BC-Insight] Failure to Generate ABI Binding in Golang
* \#37351 \[BC-Insight] Resubscribe Deadlocks When Unsubscribing Within An Unblock Channel
* \#38766 \[BC-Insight] Nil Pointer Dereference Panics in encodePayload() of Blob Tx’s Encoding
* \#38807 \[BC-Low] DoS any reth node via ban logic exploit
* \#37442 \[BC-Insight] Potential Address Collision with Precompile Contract During Contract Deployment
* \#37462 \[BC-Low] Invalid RLP decoding for single bytes
* \#37483 \[BC-Insight] There is a trace discrepancy for Nethermind when handling EOF from PUSH opcode
* \#38850 \[BC-Low] Remote P2P OOM Crash (GetBlockHeaders) / Reth
* \#37505 \[BC-Insight] Remotely spamming 1 byte leads to full peer removal and desync in both execution and consensus clients
* \#37568 \[BC-Insight] Missing Specification Logic
* \#38894 \[BC-Low] Missing expiration check for Pong and Neighbors packets and not refreshing the endpoint proof
* \#38275 \[BC-Low] Evil-client P2P headers-traversal leads to D/DoS and total peer removal
* \#37300 \[BC-Insight] Incorrect Encoding of Negative \*big.Int Values in MakeTopics
* \#38277 \[BC-Insight] Potential Out-of-Range Panic in \`UnmarshalJSON()\` of \`HexOrDecimal256\`
* \#38908 \[BC-Insight] Missing Failed Subcalls in Erigon Tracers When Encountering \`ErrInsufficientBalance\` Error
* \#38958 \[BC-Low] EELS cant handle overflow gas calculation in modexp precompile
* \#38828 \[BC-Low] Decode RLP of Legacy Transaction Allows Tailing Bytes
* \#39018 \[BC-Insight] Rate Limiting Under-Specification and Consequences
* \#37113 \[BC-Low] A potential out-of-range panic has been discovered in the Ethereum client Erigon ( <https://github.com/erigontech/erigon> ), though it does not seem to be exploitable at this moment du...
* \#38459 \[BC-Low] erigon remote DoS
* \#37246 \[BC-Low] lodestar snappy checksum issue
* \#38733 \[BC-Medium] nibmus-eth2 remote crash
* \#38920 \[BC-Medium] teku remote DoS
* \#37148 \[BC-Insight] \`wantedPeerDials()\` branch will never be executed
* \#38557 \[BC-Insight] Function \`IsPush()\` Misses Opcode PUSH0
* \#38427 \[BC-Low] Discrepancy in Intrinsic Gas Calculation between Txpool and EVM Execution
* \#37210 \[BC-Insight] Missing Check of HTTP Batch Response Length
* \#38902 \[BC-Low] No check on the maximum size of the encoded ENR on ENR\_RESPONSE packet
* \#37350 \[BC-Insight] \`null\` Is Not Unmarshalled Correctly Into json.RawMessage
* \#37104 \[BC-Insight] Reth RPC is vulnerable to DNS rebinding attacks
* \#38948 \[BC-Low] lighthouse remote DoS
* \#38015 \[BC-Insight] Violation of EIP-2681 in Create Transaction
* \#37186 \[BC-Insight] Missing Validation for Fixed-Size bytes Types in ABI Parsing
* \#38319 \[BC-Insight] Edge case difference for GETH and NETHERMIND when calculating memory expansion gas
* \#37153 \[BC-Insight] Malicious validator can bring down honest nodes
* \#38278 \[BC-Low] Potential DoS to Mempool Due to Missing Gas Limit Check
* \#38318 \[BC-Low] nimbus-eth2: Gossipsub misconfiguration allows malicious peers gossip malformed data without penalization
* \#37577 \[BC-Insight] \`tx.origin\` Usage in Group Management Contract Allows Phishing Attack for Unauthorized Actions
* \#38146 \[BC-Medium] nimbus-eth2 remote crash

</details>
