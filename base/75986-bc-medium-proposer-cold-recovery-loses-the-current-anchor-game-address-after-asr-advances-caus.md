# 75986 bc medium proposer cold recovery loses the current anchor game address after asr advances causing subsequent multiproof games to revert

**Submitted on May 2nd 2026 at 03:13:44 UTC by @Diavol0 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75986
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief/Intro

The in-scope Rust proposer recovers its latest on-chain progress from `AnchorStateRegistry.getAnchorRoot()`, but that call returns only the current output root and L2 block number. After the ASR has advanced to an `anchorGame`, the proposer still uses the ASR address as the parent sentinel during a cold recovery. That sentinel is valid only for the first game from the immutable starting anchor. The next `AggregateVerifier` game must instead use the current anchor game's address as `parentAddress`, so a restarted or cache-reset proposer can repeatedly submit transactions that revert and fail to advance proposals.

### Vulnerability Details

The affected primary asset is the off-chain proposer in `base/base` at `crates/proof/proposer/src/pipeline.rs`. The on-chain integration point is the in-scope multiproof game in `src/multiproof/AggregateVerifier.sol`.

Multiproof games bind the parent game address into `extraData`:

```rust
pub fn encode_extra_data(
    l2_block_number: u64,
    parent_address: Address,
    intermediate_roots: &[B256],
) -> Bytes {
    let mut data = vec![0u8; 52 + 32 * intermediate_roots.len()];
    data[..32].copy_from_slice(&U256::from(l2_block_number).to_be_bytes::<32>());
    data[32..52].copy_from_slice(parent_address.as_slice());
    ...
}
```

`AggregateVerifier.initializeWithInitData()` treats `parentAddress() == address(ANCHOR_STATE_REGISTRY)` as the special "first game" case:

```solidity
if (parentAddress() != address(ANCHOR_STATE_REGISTRY)) {
    IDisputeGame parentGame = IDisputeGame(parentAddress());
    if (!_isValidGame(parentGame)) revert InvalidParentGame();

    startingOutputRoot = Proposal({
        l2SequenceNumber: parentGame.l2SequenceNumber(), root: Hash.wrap(parentGame.rootClaim().raw())
    });
} else {
    startingOutputRoot = ANCHOR_STATE_REGISTRY.getStartingAnchorRoot();
}

if (l2SequenceNumber() != startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL) {
    revert UnexpectedBlockNumber(startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL, l2SequenceNumber());
}
```

This means the ASR address sentinel is correct only when the new game starts from `getStartingAnchorRoot()`. Once ASR has advanced to a real `anchorGame`, the next game must use that `anchorGame` address as its parent. If it uses the ASR address again, the contract compares the new game against the immutable starting anchor block and reverts with `UnexpectedBlockNumber`.

The Rust proposer loses the required address during cold recovery. `recover_latest_state()` first reads the current anchor root:

```rust
let anchor = self
    .anchor_registry
    .get_anchor_root()
    .await
    .map_err(|e| ProposerError::Contract(format!("get_anchor_root failed: {e}")))?;
```

However, the Rust `AnchorStateRegistryClient` interface only exposes `get_anchor_root()`, which returns the root and L2 block number. It does not expose the Solidity `anchorGame()` address. When the cache is absent or invalid, the recovery start state is built with the ASR address sentinel:

```rust
let start = match cache.as_ref() {
    Some(cached) if tip_still_valid(cached) && count > cached.game_count => {
        cached.state
    }
    _ => RecoveredState {
        parent_address: self.config.driver.anchor_state_registry_address,
        output_root: anchor.root,
        l2_block_number: anchor.l2_block_number,
    },
};
```

The forward walk then looks for the next game by UUID using this `parent_address`:

```rust
let extra_data =
    encode_extra_data(expected_block, parent_address, &intermediate_root_vec);

let lookup =
    self.factory_client.games(game_type, canonical_root, extra_data).await?;
```

Correct child games created after the ASR advanced are registered under `extraData` containing `parentAddress = address(anchorGame)`, not `parentAddress = address(AnchorStateRegistry)`. A cold proposer therefore cannot discover them. If no game is found, the same ASR sentinel is passed into the proposal submission path:

```rust
self.output_proposer.propose_output(aggregate_proposal, parent_address, &intermediate_roots).await
```

`ProposalSubmitter` encodes that parent address into `createWithInitData()` calldata:

```rust
let extra_data = encode_extra_data(l2_block_number, parent_address, intermediate_roots);
let calldata =
    encode_create_calldata(self.game_type, proposal.output_root, extra_data, proof_data);
```

The result is a production-reachable liveness failure after the ASR has advanced and the proposer later performs a full recovery, for example after a restart, cold deployment, crash recovery, or cache reset. A continuously running proposer whose in-memory cache still contains the current parent game may avoid this path, so the issue is intentionally described as a recovery/liveness bug rather than an unconditional halt.

### Impact Details

After ASR advances to a non-starting `anchorGame`, a cold or cache-reset instance of this proposer implementation cannot recover the current parent game address from chain. It can repeatedly build the next proposal with `parentAddress = AnchorStateRegistry`, and the on-chain game initialization reverts because the proposal block is checked against `getStartingAnchorRoot()` instead of the current anchor game.

This prevents the in-scope proposer from advancing multiproof proposals after normal anchor advancement unless the implementation is fixed or the missing parent address is supplied out of band. It can also cause repeated failed L1 transactions and operational intervention during recovery. The report does not demonstrate direct theft, proof forgery, or permanent fund loss, so the impact is classified conservatively as Medium griefing/availability damage.

### References

* `crates/proof/proposer/src/pipeline.rs`: `recover_latest_state()` uses `get_anchor_root()` and falls back to `anchor_state_registry_address` on full recovery.
* `crates/proof/proposer/src/pipeline.rs`: `forward_walk()` looks up games by UUID using `encode_extra_data(..., parent_address, ...)`.
* `crates/proof/proposer/src/output_proposer.rs`: `ProposalSubmitter::propose_output()` encodes the recovered parent address into `createWithInitData()` calldata.
* `crates/proof/contracts/src/anchor_state_registry.rs`: Rust ASR binding exposes `getAnchorRoot()` but not `anchorGame()`.
* `src/dispute/AnchorStateRegistry.sol`: `getAnchorRoot()` returns the current anchor root/block, and `anchorGame` is a separate public state variable.
* `src/multiproof/AggregateVerifier.sol`: `initializeWithInitData()` uses `getStartingAnchorRoot()` when `parentAddress()` is the ASR address.

## Proof of Concept

{% stepper %}
{% step %}

### Rust proposer recovery PoC

Apply the following additions to `crates/proof/proposer/src/pipeline.rs` inside the existing `#[cfg(test)] mod tests` block.

Import `GameAtIndex` with the other contract test types:

```rust
use base_proof_contracts::{AnchorRoot, GameAtIndex};
```

Add the non-zero ASR sentinel constant near the other test constants:

```rust
const TEST_ASR_ADDRESS: Address = Address::repeat_byte(0xA5);
```

Add the test:

```rust
#[tokio::test(flavor = "current_thread", start_paused = true)]
async fn test_recovery_from_advanced_anchor_uses_asr_sentinel_instead_of_anchor_game() {
    let anchor_block = TEST_BLOCK_INTERVAL;
    let anchor_root = B256::repeat_byte(0xA1);
    let anchor_game = proxy_addr(0);

    let child_block = anchor_block + TEST_BLOCK_INTERVAL;
    let child_root = B256::repeat_byte(0xB2);
    let child_intermediate_roots = vec![child_root];
    let child_extra_data =
        encode_extra_data(child_block, anchor_game, &child_intermediate_roots);

    let mut factory = MockDisputeGameFactory::with_games(vec![GameAtIndex {
        game_type: TEST_GAME_TYPE,
        timestamp: 1,
        proxy: anchor_game,
    }]);
    factory.game_count_override = Some(2);
    factory.uuid_games.insert((TEST_GAME_TYPE, child_root, child_extra_data), proxy_addr(1));

    let cancel = CancellationToken::new();
    let l1 = Arc::new(MockL1 { latest_block_number: TEST_L1_BLOCK_NUMBER });
    let l2 = Arc::new(MockL2 { block_not_found: true, canonical_hash: None });
    let prover: Arc<dyn ProverClient> =
        Arc::new(MockProver { delay: MOCK_PROVER_DELAY, block_interval: TEST_BLOCK_INTERVAL });
    let rollup = Arc::new(MockRollupClient {
        sync_status: test_sync_status(0, B256::ZERO),
        output_roots: HashMap::from([(child_block, child_root)]),
        max_safe_block: None,
    });
    let anchor_registry = Arc::new(MockAnchorStateRegistry {
        anchor_root: AnchorRoot { root: anchor_root, l2_block_number: anchor_block },
    });

    let pipeline = ProvingPipeline::new(
        PipelineConfig {
            max_parallel_proofs: 1,
            max_retries: 1,
            recovery_scan_concurrency: 8,
            tee_prover_registry_address: None,
            driver: DriverConfig {
                game_type: TEST_GAME_TYPE,
                block_interval: TEST_BLOCK_INTERVAL,
                intermediate_block_interval: TEST_BLOCK_INTERVAL,
                anchor_state_registry_address: TEST_ASR_ADDRESS,
                ..Default::default()
            },
        },
        prover,
        l1,
        l2,
        rollup,
        anchor_registry,
        Arc::new(factory),
        Arc::new(MockAggregateVerifier::default()),
        Arc::new(MockOutputProposer),
        cancel,
    );

    let mut cache: Option<CachedRecovery> = None;
    let state = pipeline.recover_latest_state(&mut cache).await.unwrap();

    assert_eq!(state.l2_block_number, anchor_block);
    assert_eq!(state.output_root, anchor_root);
    assert_eq!(
        state.parent_address, TEST_ASR_ADDRESS,
        "recovery keeps the ASR sentinel instead of the current anchor game address"
    );
    assert_ne!(state.parent_address, anchor_game);
}
```

Run:

```bash
CARGO_ENCODED_RUSTFLAGS='' cargo test --manifest-path /Users/Juxin.Gao/solidity/imm/base-0.8.0-rc.28/Cargo.toml -p base-proposer test_recovery_from_advanced_anchor_uses_asr_sentinel_instead_of_anchor_game -- --nocapture
```

Observed result:

```
running 1 test
test pipeline::tests::test_recovery_from_advanced_anchor_uses_asr_sentinel_instead_of_anchor_game ... ok

test result: ok. 1 passed; 0 failed
```

The important assertion is that recovery returns the non-zero ASR sentinel, not the real `anchorGame` address, even though the mock factory contains both the current anchor game and a valid next child keyed by the real parent.
{% endstep %}

{% step %}

### Contract-side revert PoC

Create `test/multiproof/AdvancedAnchorParentSentinelPoC.t.sol` with the following contents:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { Claim, Hash } from "src/dispute/lib/Types.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";

import { BaseTest } from "./BaseTest.t.sol";

contract AdvancedAnchorParentSentinelPoCTest is BaseTest {
    function testAsrSentinelCannotBeUsedAsParentAfterAnchorAdvanced() public {
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim parentRoot = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "parent")));
        bytes memory parentProof = _generateProof("parent-proof", AggregateVerifier.ProofType.TEE);

        AggregateVerifier parentGame = _createAggregateVerifierGame(
            TEE_PROVER, parentRoot, currentL2BlockNumber, address(anchorStateRegistry), parentProof
        );

        vm.warp(block.timestamp + 7 days);
        parentGame.resolve();
        vm.warp(block.timestamp + 1);
        parentGame.closeGame();

        (Hash anchorRoot, uint256 anchorBlock) = anchorStateRegistry.getAnchorRoot();
        assertEq(anchorRoot.raw(), parentRoot.raw());
        assertEq(anchorBlock, BLOCK_INTERVAL);
        assertEq(address(anchorStateRegistry.anchorGame()), address(parentGame));

        uint256 nextBlock = currentL2BlockNumber + BLOCK_INTERVAL;
        Claim childRoot = Claim.wrap(keccak256(abi.encode(nextBlock, "child")));
        bytes memory childProof = _generateProof("child-proof", AggregateVerifier.ProofType.TEE);

        vm.expectRevert(
            abi.encodeWithSelector(AggregateVerifier.UnexpectedBlockNumber.selector, BLOCK_INTERVAL, nextBlock)
        );
        _createAggregateVerifierGame(TEE_PROVER, childRoot, nextBlock, address(anchorStateRegistry), childProof);

        AggregateVerifier childGame =
            _createAggregateVerifierGame(TEE_PROVER, childRoot, nextBlock, address(parentGame), childProof);
        assertEq(childGame.parentAddress(), address(parentGame));
        assertEq(childGame.l2SequenceNumber(), nextBlock);
    }
}
```

Run:

```bash
forge test --match-path test/multiproof/AdvancedAnchorParentSentinelPoC.t.sol -vvv
```

Observed result:

```
Ran 1 test for test/multiproof/AdvancedAnchorParentSentinelPoC.t.sol:AdvancedAnchorParentSentinelPoCTest
[PASS] testAsrSentinelCannotBeUsedAsParentAfterAnchorAdvanced()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The test first proves that ASR has advanced to `parentGame`. It then shows that the next game reverts when the parent is the ASR sentinel, exactly matching the parent address that the cold proposer recovery returns. The control path using `address(parentGame)` succeeds.
{% endstep %}
{% endstepper %}

### Recommended Mitigation

Recover and preserve the actual current anchor game address whenever ASR has advanced.

Possible fixes:

* Extend the Rust `AnchorStateRegistry` binding to read `anchorGame()` in addition to `getAnchorRoot()`.
* Use `parentAddress = address(AnchorStateRegistry)` only when `anchorGame == address(0)`.
* When `anchorGame != address(0)`, use `anchorGame` as the recovered `parent_address` before starting the forward walk.
* Add a pre-submission guard that rejects `parentAddress == AnchorStateRegistry` when the recovered anchor block differs from `getStartingAnchorRoot().l2SequenceNumber`.
* Add a cross-repository recovery test that advances ASR, drops the proposer cache, and verifies that the next proposal is encoded with `parentAddress = anchorGame`.
