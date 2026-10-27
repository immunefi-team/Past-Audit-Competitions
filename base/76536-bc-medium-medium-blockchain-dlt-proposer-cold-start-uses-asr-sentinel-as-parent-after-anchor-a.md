# 76536 bc medium medium blockchain dlt proposer cold start uses asr sentinel as parent after anchor advancement causing aggregateverifier initializewithinitdata to revert unexpectedblocknumber&#x20;

**Submitted on May 4th 2026 at 19:21:32 UTC by @Devomacky for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76536
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief / Intro

The Base Azul proposer cold-start path reconstructs its next game from `AnchorStateRegistry.getAnchorRoot()` but encodes `parent_address = AnchorStateRegistry` in `extraData`. Once `AnchorStateRegistry.anchorGame` has been advanced by a valid finalized game, `getAnchorRoot()` returns the live advanced anchor while `AggregateVerifier.initializeWithInitData()` interprets `parentAddress() == ANCHOR_STATE_REGISTRY` as the genesis/sentinel case and reads `getStartingAnchorRoot()` instead. The result is a deterministic `UnexpectedBlockNumber(100, 200)` revert on every proposer cold-start after the anchor has advanced. This is a Base layer-2 network-code bug that produces unintended smart contract behavior with no concrete funds at direct risk.

## Vulnerability Details

Selected asset:

```
https://github.com/base/base/tree/v0.8.0-rc.28
Blockchain/DLT
```

Selected impact:

```
A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
```

The root cause is a cross-component semantic mismatch between the off-chain proposer and the on-chain `AggregateVerifier`.

In the selected `base/base@v0.8.0-rc.28` asset, `crates/proof/proposer/src/pipeline.rs::recover_latest_state` reads the live anchor:

```rust
let anchor = self.anchor_registry.get_anchor_root().await?;
```

On the cold-start branch it then returns a recovered state with:

```rust
parent_address: self.config.driver.anchor_state_registry_address,
output_root: anchor.root,
l2_block_number: anchor.l2_block_number,
```

This combines a live advanced anchor block with the ASR sentinel parent address.

In the supporting in-scope contract code, `AggregateVerifier.initializeWithInitData()` treats `parentAddress() == ANCHOR_STATE_REGISTRY` as the first-game sentinel and reads `getStartingAnchorRoot()`:

```solidity
if (parentAddress() != address(ANCHOR_STATE_REGISTRY)) {
    IDisputeGame parentGame = IDisputeGame(parentAddress());
    if (!_isValidGame(parentGame)) revert InvalidParentGame();
    startingOutputRoot = Proposal({
        l2SequenceNumber: parentGame.l2SequenceNumber(),
        root: Hash.wrap(parentGame.rootClaim().raw())
    });
} else {
    startingOutputRoot = ANCHOR_STATE_REGISTRY.getStartingAnchorRoot();
}

if (l2SequenceNumber() != startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL) {
    revert UnexpectedBlockNumber(
        startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL,
        l2SequenceNumber()
    );
}
```

`AnchorStateRegistry.getStartingAnchorRoot()` and `getAnchorRoot()` are intentionally different after an anchor game has been accepted:

```solidity
function getStartingAnchorRoot() external view returns (Proposal memory) {
    return startingAnchorRoot;
}

function getAnchorRoot() public view returns (Hash, uint256) {
    if (address(anchorGame) == address(0)) {
        return (startingAnchorRoot.root, startingAnchorRoot.l2SequenceNumber);
    }
    return (Hash.wrap(anchorGame.rootClaim().raw()), anchorGame.l2SequenceNumber());
}
```

After `anchorGame` is set, the proposer reads the advanced block from `getAnchorRoot()`, computes `advanced + BLOCK_INTERVAL`, and dispatches a game whose parent is still encoded as the ASR sentinel. The contract then compares that advanced block against `getStartingAnchorRoot().l2SequenceNumber + BLOCK_INTERVAL` and reverts.

The Rust binding in `crates/proof/contracts/src/anchor_state_registry.rs` exposes `getAnchorRoot()` but does not expose `anchorGame()`. Therefore the proposer cannot recover the live anchor-game proxy address and use it as the parent on cold-start. There is no retry path in `forward_walk` that switches from the ASR sentinel to the actual anchor-game address after the revert.

## Reachability

The trigger condition is `ASR.anchorGame != address(0)`.

That condition is reachable through `AnchorStateRegistry.setAnchorState(game)`, which is a public permissionless entrypoint in `base/contracts@v8.1.0`. Only one accepted finalized DEFENDER\_WINS game is needed for the ASR live anchor to diverge from the starting anchor.

The condition is also amplified by mid-contest PR `base/base#2372`, which automates this same permissionless action. The PR description states that the challenger sends a permissionless `setAnchorState(game)` transaction to the ASR contract after a game resolves. That makes anchor advancement a routine outcome of normal successful game finalization, not an exotic attacker-only path.

The second trigger is a proposer cold-start or pipeline reset. This is a normal operational event:

* first process start after deployment
* binary redeploy
* host/container restart
* OOM/panic restart
* task panic paths that call `PipelineState::reset`
* `SubmitOutcome::RootMismatch`, which resets the pipeline when canonical output changed between proof generation and submission
* repeated proof failures after max retries

The bug relies on restarts happening, not on restarts failing to happen. This matters because the program's own Known Vulnerabilities material states that some proposer configuration changes require redeploy. Cold starts are therefore expected operational behavior.

## Impact Details

The impact is exactly:

```
A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
```

The affected layer-2 network code is the Base proposer recovery pipeline and the Rust ASR binding in the selected `base/base@v0.8.0-rc.28` asset.

The unintended smart-contract behavior is the deterministic `AggregateVerifier.UnexpectedBlockNumber(expected, actual)` revert when the proposer attempts to create the next game after a valid anchor advancement.

The PoC observes:

```
selector: 0x087b2d76
selector("UnexpectedBlockNumber(uint256,uint256)"): 0x087b2d76
expected: 100
actual: 200
```

There are no concrete funds at direct risk in this report. The revert occurs during `AggregateVerifier.initializeWithInitData()` before the proposal is accepted and before the game bond can become protocol-risky. This report does not claim bridge drain, fund theft, chain split, or permanent fund freeze.

The practical effect is proposer liveness failure after a valid anchor advancement: every cold-start repeats the same construction and hits the same revert until the client code is patched to use the actual `anchorGame()` address as parent when `anchorGame != address(0)`.

## Why The Common Closure Arguments Do Not Apply

**Not out of scope:** the primary bug is in `base/base@v0.8.0-rc.28`, specifically the proposer cold-start recovery path and Rust ASR binding. The contract code is cited only because it is the in-scope smart contract boundary where the unintended behavior is observed.

**Not a unit-test PoC:** the PoC is a `forge script` run through `forge script`, not a `forge test` unit test. It inherits from `forge-std/Script.sol`, has a `run()` entrypoint, and exercises the full deployed-contract lifecycle locally against v8.1.0 source bytecode.

**Not dependent on invalid TEE/ZK proofs:** the revert occurs before proof verification. `_verifyProof()` is not reached.

**Not dependent on Base failing to blacklist or retire an invalid game:** there is no invalid proposal to blacklist or retire. The failing proposal reverts before it exists as a valid game. Guardian actions do not clear `anchorGame` or rewrite `startingAnchorRoot`; recovery requires code changes.

**Not duplicate of PR #2372:** PR #2372 is the amplifier. It causes the challenger to perform the anchor update that makes cold-start fail. It does not patch `recover_latest_state`, the Rust ASR binding, or `AggregateVerifier`'s starting/current anchor semantic mismatch.

The audit competition mid-contest changelog states verbatim: "If a new bug is introduced by their fix then it is valid for a reward." This finding is exactly that case: PR #2372 introduces the precondition (`anchorGame != address(0)` becoming routine via permissionless `setAnchorState`) that makes the cold-start path deterministically revert.

**Not a known issue:** grep across the provided audits, Optimism reviews, and Known Vulnerabilities material found no disclosure of the cold-start proposer / `getStartingAnchorRoot()` / `UnexpectedBlockNumber` interaction. Adjacent ASR issues concern different invariants.

## Recommended Remediation

Patch the Rust ASR binding to expose `anchorGame()`:

```rust
sol! {
    #[sol(rpc)]
    interface IAnchorStateRegistry {
        function anchorGame() external view returns (address);
        function getAnchorRoot() external view returns (bytes32 root, uint256 l2SequenceNumber);
    }
}
```

Then patch `recover_latest_state` so cold-start uses the actual anchor game address if present:

```rust
let anchor = self.anchor_registry.get_anchor_root().await?;
let anchor_game = self.anchor_registry.anchor_game().await?;

let parent_address = if anchor_game.is_zero() {
    self.config.driver.anchor_state_registry_address
} else {
    anchor_game
};
```

With `parent_address = anchorGame`, `AggregateVerifier.initializeWithInitData()` enters the non-ASR branch, reads the parent game's `l2SequenceNumber()`, and the block-number check matches the proposer's construction.

## References

* Secret Gist with supporting artifacts: `https://gist.github.com/dsantoreis/95f4751f6c344f172eacafb4ead6473d`
* Selected asset: `https://github.com/base/base/tree/v0.8.0-rc.28`
* Supporting contract asset: `https://github.com/base/contracts/tree/v8.1.0/src/multiproof`
* PoC script: `ColdStartProposerCannotResume.s.sol`
* PoC output: `coldstart_run.log`
* Code evidence: `coldstart_code_evidence.txt`
* `SUBMISSION.md` SHA256: `9d9d0651606ed77ef93cca602adbdd6f90c8d6a89af85815b46b75c6be5cbe8e`
* `ColdStartProposerCannotResume.s.sol` SHA256: `e465bec08291d612f633eb3b7535406482cf91af10f53f3cdb787c64bf944895`
* `coldstart_run.log` SHA256: `82bb813c2081d163f84737e15f945d9766dbd23c2f020c54829a61f58614bb68`
* `coldstart_code_evidence.txt` SHA256: `175bcb52b07c8319cec46a2aa2b11787bc09c7d08f045bb96744d541896559d3`

## Link to Proof of Concept

<https://gist.github.com/dsantoreis/95f4751f6c344f172eacafb4ead6473d>

## Proof of Concept

This PoC runs only locally. It does not broadcast transactions to Sepolia or mainnet. It compiles the in-scope `base/contracts@v8.1.0` source, deploys the relevant contracts locally, advances ASR through the public `setAnchorState()` path, and then replays the proposer cold-start dispatch shape produced by `base/base@v0.8.0-rc.28`.

Secret Gist with supporting artifacts:

```
https://gist.github.com/dsantoreis/95f4751f6c344f172eacafb4ead6473d
```

## Environment

Repository:

```bash
cd /root/base-azul-audit/contracts-v8.1.0
```

Compiler:

```
Solc 0.8.15
```

PoC file:

```
script/ColdStartProposerCannotResume.s.sol
```

The script is a forge script, not a unit test:

```
forge-std/Script.sol
run()
forge script ...
```

It is not:

```
forge-std/Test.sol
test_*
forge test
```

## Run Command

```bash
forge script script/ColdStartProposerCannotResume.s.sol:ColdStartProposerCannotResume -vv | tee /root/base-azul-audit/five-hunts-20260504/coldstart/coldstart_run.log
```

## Observed Output

The clean reproduced output is:

```
Warning: Found unknown `additional_compiler_profiles` config key in section `invariant` defined in foundry.toml.
Warning: Found unknown `compilation_restrictions` config key in section `invariant` defined in foundry.toml.
Compiling 1 files with Solc 0.8.15
Solc 0.8.15 finished in 3.90s
Compiler run successful!
Script ran successfully.
Gas used: 12057071

== Logs ==
  == Step 1. starting anchor (l2SequenceNumber) = 0
  == Step 1. anchorGame address                 = 0x0000000000000000000000000000000000000000
  == Step 2(a). game1 created at                  = 0xB03b98b4cD0F8C68167D5EAc0203e91B655c846c
  ==            game1.l2SequenceNumber()          = 100
  ==            game1.parentAddress() == ASR      = true
  == Step 2(c). game1.resolve() returned status   = 2
  == Step 2(e). setAnchorState(game1) succeeded.
  == Step 2(e). ASR.anchorGame                     = 0xB03b98b4cD0F8C68167D5EAc0203e91B655c846c
  == Step 2(e). ASR.getAnchorRoot().l2SequenceNumber =  100
  == Step 2(e). ASR.getStartingAnchorRoot().l2SeqNum =  0
  == Step 3. proposer cold-start dispatch construction
  ==          parent_address (== ASR proxy)            = 0x72bCCcCAfEA3A7a98A9cF8Fa980f3E4ef3244fC4
  ==          expected_block (anchor + interval)       = 200
  == Step 4. Revert observed.
  0x087b2d76
             expected selector for UnexpectedBlockNumber:
  0x087b2d76
             contract expected (= startingAnchor + BLOCK_INTERVAL) = 100
             contract actual   (= advancedAnchor + BLOCK_INTERVAL) = 200
  ============================================================
    COLD-START LIVENESS BUG REPRODUCED.
    Proposer cold-start after the on-chain effect of PR #2372
    cannot create a valid next AggregateVerifier game because
    the contract uses getStartingAnchorRoot() while the
    Rust client only exposes getAnchorRoot(). Every cold-start
    reverts with UnexpectedBlockNumber until a code change
    is made.
  ============================================================
```

## Step-By-Step Explanation

{% stepper %}
{% step %}

## The script deploys the relevant v8.1.0 contracts locally

* `DisputeGameFactory`
* `AnchorStateRegistry`
* `DelayedWETH`
* `AggregateVerifier`
  {% endstep %}

{% step %}

## The script confirms the starting anchor state

```
starting anchor l2SequenceNumber = 0
anchorGame = 0x0000000000000000000000000000000000000000
```

{% endstep %}

{% step %}

## The script creates the first valid `AggregateVerifier` game

```
l2SequenceNumber = 100
parentAddress = AnchorStateRegistry
```

This succeeds because `AggregateVerifier.initializeWithInitData()` reads `getStartingAnchorRoot()` and expects:

```
0 + BLOCK_INTERVAL = 100
```

{% endstep %}

{% step %}

## The script resolves the first game as `DEFENDER_WINS`

The script calls the public ASR entrypoint:

```solidity
anchorStateRegistry.setAnchorState(IDisputeGame(address(game1)));
```

After this call:

```
ASR.getAnchorRoot().l2SequenceNumber = 100
ASR.getStartingAnchorRoot().l2SequenceNumber = 0
```

This reproduces the on-chain state that exists after the anchor has been advanced by a finalized DEFENDER\_WINS game.
{% endstep %}

{% step %}

## The script replays the proposer cold-start construction

```
parent_address = AnchorStateRegistry
parent_block = ASR.getAnchorRoot().l2SequenceNumber = 100
next block = 100 + BLOCK_INTERVAL = 200
```

{% endstep %}

{% step %}

## The script submits the next game

The next game is submitted through the factory with `parentAddress == ASR` and `l2SequenceNumber == 200`.
{% endstep %}

{% step %}

## `AggregateVerifier.initializeWithInitData()` interprets the ASR parent as the first-game sentinel

It reads `getStartingAnchorRoot()` instead of `getAnchorRoot()`.

The contract expects:

```
getStartingAnchorRoot().l2SequenceNumber + BLOCK_INTERVAL = 0 + 100 = 100
```

But the proposer has constructed:

```
getAnchorRoot().l2SequenceNumber + BLOCK_INTERVAL = 100 + 100 = 200
```

{% endstep %}

{% step %}

## The call reverts with `UnexpectedBlockNumber(100, 200)`

The selector is verified:

```
0x087b2d76 == bytes4(keccak256("UnexpectedBlockNumber(uint256,uint256)"))
```

{% endstep %}
{% endstepper %}

## Why This PoC Is Not Self-Referential

The PoC does not compare two locally invented encodings and does not stop at a unit-level assertion.

It performs the full contract lifecycle locally:

1. deploys v8.1.0 source contracts
2. creates a valid game
3. resolves that game
4. advances ASR through the public `setAnchorState()` path
5. constructs the next game using the same parent/block shape as the proposer cold-start branch
6. calls `DisputeGameFactory.createWithInitData`
7. observes the real `AggregateVerifier.initializeWithInitData()` revert

The only mocked component is `MockVerifier`, which is shipped by Base in:

```
src/multiproof/mocks/MockVerifier.sol
```

Base's own `test/multiproof/BaseTest.t.sol` uses the same mock to stand in for proof verification in `AggregateVerifier` tests. The revert occurs before proof verification, so proof validity is not relevant to the demonstrated failure.

## Artifacts

```
SUBMISSION.md
SHA256: 9d9d0651606ed77ef93cca602adbdd6f90c8d6a89af85815b46b75c6be5cbe8e

ColdStartProposerCannotResume.s.sol
SHA256: e465bec08291d612f633eb3b7535406482cf91af10f53f3cdb787c64bf944895

coldstart_run.log
SHA256: 82bb813c2081d163f84737e15f945d9766dbd23c2f020c54829a61f58614bb68

coldstart_code_evidence.txt
SHA256: 175bcb52b07c8319cec46a2aa2b11787bc09c7d08f045bb96744d541896559d3
```

All artifacts are OpenTimestamp-stamped. The `.ots` files and manifest are included in the supporting Gist.
