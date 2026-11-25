# \#98825 [SC-High] Permissionless liquidation order leaves an owner-cleared degraded latch and freezes a mature third-party deposit

**Submitted on Nov 10th 2026 at 12:23:55 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98825
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

# Brief / Intro

`executeLiquidationBatch` lets an ordinary keeper choose account order. Each
successful liquidation commits before the next item. A constructed opposing
pair is liquidatable at one mark in both orders, but the order is
security-relevant:

- safe control `LONG -> SHORT` never enters degraded mode;
- harmful `SHORT -> LONG` makes the first item cross `E < L` and set the
  latch; the second item restores the exact incumbent book, but
  `_enterDegradedModeIfInsolvent` returns immediately because the bit is
  already true.

Both coalition positions then cash out and end flat. The harmful branch still
blocks the whole LP epoch. A distinct depositor's already mature
`20,000,000 USDG` Junior request remains pending, unclaimable, and
nonrefundable at the first ordinary settlement checkpoint and again `86,401`
seconds later. The byte-comparable control makes the full request claimable at
the first checkpoint.

This report does not claim that a sticky owner-cleared containment latch is
itself accidental. Plether's documentation and tests deliberately require an
owner clear. The defect is the unprivileged order-dependent composition: a
permissionless caller can choose which of two otherwise equivalent terminal
paths leaves the global containment bit set against an unrelated mature
depositor after the caller's accounts have fully exited.

# Vulnerability Details

## 1. A permissionless keeper chooses sequential batch order

`OrderRouter.executeLiquidationBatch` accepts an account array from the
caller. `OrderLiquidationBatchLogic.executeLiquidationBatch` processes the
array sequentially through independently caught calls, preserving successful
earlier items:

- `packages/perps/src/OrderRouter.sol:500-520`;
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:738-797`.

No owner, guardian, victim, or incumbent-account authority is needed. H378
uses four distinct ordinary actors for the victim, LONG owner, SHORT owner,
and keeper.

## 2. First-item containment can survive later solvency restoration

Each liquidation ends in
`_enterDegradedModeIfInsolvent(delta.account, 0)` at
`CfdEngine.sol:1344`. The helper starts:

```solidity
if (degradedMode) {
    return;
}
```

at `CfdEngine.sol:1351-1368`. Thus the first item can set the latch when its
intermediate post-item state has `E < L`. A later item may remove the
coalition's remaining liability and restore raw solvency, but the already-true
branch performs no normalization.

Only `clearDegradedMode()` can clear it, and that function is owner-only:
`CfdEngine.sol:655-666`.

## 3. The latch blocks deposits before settlement can heal anything

`HousePool._settleLpEpoch` calls `_requireWithdrawalsLive` before
reconciliation, redemptions, or deposits
(`HousePool.sol:823-843`). That helper reverts solely when the Engine snapshot
has `degradedMode == true` (`HousePool.sol:1864-1869`). It does not separate
cash-out processing from a cash-in deposit.

The Junior victim is already mature. Its cancellation attempt reverts
`TrancheVault__DepositEpochAlreadyActive()` under
`TrancheVault.sol:826-860`. The depositor therefore cannot activate or refund
the escrow through an ordinary victim action while global settlement is
blocked.

# Exact-Current H378 Causal Proof

The attached one-command proof starts two separate fresh local Anvil forks at:

```text
source commit a82d4fefb02768e5b3bddcc97dee371eee24fea8
source tree   d88b5f9fca750cd69197b33e99a5189d3f1eccf2
chain         421614
block         326291776
block hash    0xe3adca8a8fdc888cba9a3e7a41cb157b2580f21a21711168653eca584dd8399c
manifest sha  584cdacada35f8225d2a88811edfbd9622764fe0e6da4b275ba8f33d6f8c0c74
```

It verifies all 30 v2.0.0 in-scope deployed runtime hashes. It changes no
in-scope runtime. Local `anvil_setCode` is restricted to the out-of-scope Pyth
and ArbSys dependencies, and the target child knows only a literal loopback
URL. The upstream hydration proxy blocks public `eth_call`, estimation,
trace/debug, signing, and transaction methods.

Both branches create equal `67,800,000e18` LONG and SHORT positions under the
deployed configuration. They have identical deployed state, actors, positions,
source-exact coherent marks, clocks, liquidation previews, and economics. The
only branch input difference is the account array order.

| Field | Control `LONG -> SHORT` | Harmful `SHORT -> LONG` |
|---|---:|---:|
| Pool assets after both items | `102,311,266.186296 USDG` | `102,311,266.186296 USDG` |
| LONG cash | `2,021,107.986342 USDG` | `2,021,107.986342 USDG` |
| SHORT cash | `1,755,663.877065 USDG` | `1,755,663.877065 USDG` |
| Keeper cash | `65,606.478106 USDG` | `65,606.478106 USDG` |
| Coalition position/claim/lock/reserve/order/carry | all zero | all zero |
| Total trader claims | zero | zero |
| Incumbent positions remain | true | true |
| Max incumbent directional liability | `36,169,720.828505 USDG` | `36,169,720.828505 USDG` |
| Terminal LP-price delta | `319,658.095248 USDG` | `319,658.095248 USDG` |
| Degraded bit | false | true |

The table deliberately does not claim a globally empty deployed book. The
coalition is flat and its contributions return exactly to the incumbent book;
the incumbent deployment still has open positions and nonzero directional
liability in both branches.

At `T0 = 1794176400`, both branches are outside FAD and oracle-frozen windows
with a fresh exact mark:

- control settlement succeeds; victim pending becomes zero and all
  `20,000,000 USDG` becomes claimable;
- harmful settlement returns `0x9f7adff3`
  (`HousePool__DegradedMode()`); all `20,000,000 USDG` remains pending, with
  zero claimable and refundable value.

At `T1 = 1794262801`, another fresh mark produces the same harmful result:

```text
1794262801 - 1794176400 = 86,401 seconds
```

The control's repeated settlement correctly returns
`HousePool__NoLpEpochProgress()` because its request was already settled; the
full claimable balance remains. This is not treated as a control failure.

# Impact and Severity Boundary

The candidate programme impact is High: temporary freezing of another user's
funds for at least 24 hours. Ownership is cleanly separated: the attacker
controls the two position actors and permissionless keeper; the
`20,000,000 USDG` request belongs to a distinct mature depositor.

The control proves the request would otherwise become fully claimable at T0,
so the measured interval is not ordinary epoch latency. The report claims
temporary freezing, not theft or permanent loss.

## Direct answer to intentional owner-cleared containment

The strongest adverse fact is explicit:

- `ACCOUNTING_SPEC.md:1203-1229` calls degraded mode a containment latch, says
  the owner may clear at `E >= L`, and says LP epoch settlement including
  deposit activation is blocked;
- `WHITEPAPER.md:1335-1339` says the latch deliberately does not auto-clear
  merely because a later state looks solvent;
- `PositionProtection.t.sol:752` asserts that a safety close must not silently
  clear the risk-off latch.

At H378 terminal state the canonical clear predicate is recovered, so the owner
can clear immediately. The attacker neither controls nor prevents that
transaction. Accordingly, High severity requires the otherwise valid owner
recovery not to occur for the measured 24-hour interval. The retained current
programme text does not assign a response deadline merely because monitoring
or recovery functionality exists, so manual clear does not technically erase
the measured route. It does make accepted-design treatment, response evidence,
and severity adjudication the strongest remaining closure risk.

This report therefore does not recommend blindly auto-clearing every safety
latch. The least design-conflicting remediation is to preserve a mature
depositor refund path, or introduce a deposit-only recovery phase that cannot
fund withdrawals and activates cash-in only when canonical post-activation
solvency/buffer predicates pass. A batch-end proof-based normalization can
also be considered, but only if it preserves the intended conservative
containment semantics.

# Feasibility and Economics

H378 measures:

```text
controlled capital = 4,000,000.000000 USDG
coalition cash      = 3,842,378.341513 USDG
pre-gas cost        =   157,621.658487 USDG
frozen principal   = 20,000,000.000000 USDG
ratio              = 126.886115727868194614x
```

The attack branch uses `3,545,399` gas after shared setup. The complete
24-transaction harmful construction uses `16,322,841` local gas and
`24,512` calldata bytes in retained H378 evidence. These are measurements, not
a production-dollar estimate.

The deployed scoped comparator exposes permissionless MockUSDG minting. This
proves exact-current Arbitrum Sepolia test-deployment reachability and
mechanics, not intended Arbitrum One real-USDG acquisition/liquidity, current
production TVL, a historical signed-Pyth payload, or production transaction
cost. Those are separate production-feasibility gates.

# Programme Route and PoC Compliance

The competition route refreshed on 9 November 2026 permits either the
repository Foundry suite or a local Anvil fork of the Arbitrum Sepolia v2.0.0
deployment, with
in-scope contracts unmodified and mocks/adapters limited to external
Pyth/USDG infrastructure. The packet uses the local-fork route rather than an
Immunefi-disallowed unit-test-only proof.

The packet includes:

- one fail-closed launcher;
- a standalone loopback-only target;
- a key-safe state-read-only hydration proxy;
- exact source/manifest/tool hash gates;
- a fresh-fork control and harmful branch;
- collision-safe one-shot run identity;
- exact report-inline/file byte checks, including one final LF;
- additive failure preservation.

The portable source packet executed successfully on 9 November 2026. The clean
R03 upload archive was then extracted into a new durable directory and that
exact extracted artifact executed successfully again. Both executions produced
byte-identical `control-result.json`, `attack-result.json`, and
`comparison.json`; the final archive comparison hash is
`540136ced62030f25d438f70507397f666d104cad5cb6e7df8b1775b40fcff44`.
The final proxy ledger allowed only chain id, block, and transaction-receipt
state reads, blocked both `eth_gasPrice` attempts, allowed no risky method, and
recorded zero public transactions or public-node exploit calls. The archive's
secret/path leak scan passed.

# Exact-Prior and Accepted-Design Analysis

No retained public item composes all five edges: caller-selected batch order;
first-item degraded entry; later solvency restoration with the early-return
stale bit; global LP settlement blocking; and a distinct mature,
noncancellable depositor frozen for the scoped duration.

The design overlap is nevertheless material:

- PR #55 documents permissionless keeper-selected batch order and preservation
  of earlier successes;
- PR #74 documents path-dependent latching while closes/liquidations remain
  available;
- PR #132 and current accounting docs document the sticky owner-clear policy;
- PR #62 documents degraded LP-settlement blocking and queued requests;
- PR #128 / audit LOW-28 notes that per-account previews do not simulate other
  batch items.

This is a distinct unsafe-composition argument, not a claim that its primitives
were undisclosed. The 9 November public refresh found the competition known
issues unchanged, the public code-owner `master` still at
`6909f261c7983f7651536a55df1f5fd45b50face`, and no issue or pull request newer
than H377 that discloses the exact composition. Private and concurrent findings
remain unknowable before authenticated filing-time review. Formal CVQC must
still decide whether the complete order-only composition is reportable despite
the substantial public disclosure of every primitive policy.

# One-Finding Boundary

The constructed pair, harmful/safe permutation, stale owner-cleared latch,
blanket LP-epoch gate, mature victim, temporary-freeze impact, and refund or
batch-end normalization repair all belong to PLETHER-AF-008. No other
candidate's trigger, victim, impact, economics, or remediation is relied upon.
Splitting H378 into another report would create same-root/same-impact
duplicate risk.

# Limitations and Gates

- Technical mechanism: supported by retained H378, independent S463 discovery,
  and two successful executions of the portable proof, including the exact
  freshly extracted R03 attachment.
- Programme route: public rules and known-issue list refreshed on 9 November;
  authenticated private/concurrent-knownness and form state remain operator
  gates.
- Production reachability: exact scoped test deployment supported; intended
  Arbitrum One real-asset reachability remains open.
- Severity: High temporary-freeze route supported but fragile to immediate
  owner clear and formal accepted-design adjudication.
- QC: builder execution, exact-inline, archive, field, leak, and claim checks
  are complete; formal Claude M1 CVQC remains mandatory and is not substituted
  by the builder's own adversarial review.
- Filing: no authenticated action, upload, filing, or send is authorized.

`NO_OUTCOME_ESTIMATES`.


## Link to Proof of Concept

https://gist.github.com/AFestive/f5b71f463057584c497b65a897def860

## Proof of Concept

# Proof of Concept

This is the complete validity-bearing H378 exploit and differential-comparison
code. It runs the exact listed Arbitrum Sepolia v2.0.0 contracts on two separate
fresh local Anvil forks. It verifies all 30 deployed runtime hashes before the
scenario, changes no in-scope runtime, and uses distinct ordinary actors. The
only local code replacements are external Pyth and ArbSys adapters.

The attached fail-closed launcher and state-read-only hydration proxy are
orchestration guards, not hidden exploit logic. They are bound by
`poc/POC_MANIFEST.sha256`. The target contains only
`http://127.0.0.1:29175`; the archive credential is read only by the proxy
from process memory and is excluded from target argv and logs.

## Frozen identity

- commit/tree:
  `a82d4fefb02768e5b3bddcc97dee371eee24fea8 /
  d88b5f9fca750cd69197b33e99a5189d3f1eccf2`
- chain/block/hash:
  `421614 / 326291776 /
  0xe3adca8a8fdc888cba9a3e7a41cb157b2580f21a21711168653eca584dd8399c`
- deployment manifest:
  `584cdacada35f8225d2a88811edfbd9622764fe0e6da4b275ba8f33d6f8c0c74`
- collision-safe identity: `AF008-H378-PORTABLE-R01-EXECUTION-001`

## Reproduction

Requirements are Python 3, Git, the clean recursive frozen checkout, hash-pinned
Foundry 1.5.1, hash-pinned solc 0.8.35, and an operator-supplied
archive-capable Arbitrum Sepolia endpoint. Keep the endpoint in the
`PLETHER_ARCHIVE_RPC_URL` environment variable, never argv.

After saving the inline files at their shown paths, retain the attached
`poc/run.sh`, `poc/run.py`, `poc/tools/rpc_key_proxy.py`, manifest, and
fixtures. Set `PLETHER_SOURCE_DIR`, `FOUNDRY_BIN_DIR`, `SOLC_PATH`,
`PLETHER_ARCHIVE_RPC_URL`, and
`AF008_RUN_IDENTITY=AF008-H378-PORTABLE-R01-EXECUTION-001`, keeping the
credential-bearing archive URL in the environment and out of argv and logs.
From the extracted archive root run:

```sh
./poc/run.sh --execute AF008-H378-PORTABLE-R01-EXECUTION-001
```

The launcher consumes the identity before its offline build, preserves every
failure, starts a fresh control fork and then a fresh harmful-order fork, and
performs an internal fail-closed differential comparison. Its success line is:

```text
PLETHER-AF-008 PORTABLE PROOF PASS: AF008-H378-PORTABLE-R01-EXECUTION-001
```

Then run the included independent result-file comparator:

```sh
python3 poc/compare.py poc/runs/AF008-H378-PORTABLE-R01-EXECUTION-001
```

Its expected output is:

```text
AF008_H378_DIFFERENTIAL_PASS
CONTROL_ORDER LONG_SHORT DEGRADED 0 CLAIMABLE 20000000000000
ATTACK_ORDER SHORT_LONG DEGRADED 1 PENDING 20000000000000
FREEZE_SECONDS 86401 PRE_GAS_COST 157621658487 RATIO_WAD 126886115727868194614
```

The exact R03 upload archive was freshly extracted and executed during builder
finalization. Its control, attack, and comparison result files were
byte-identical to the first portable execution; the comparison SHA-256 is
`540136ced62030f25d438f70507397f666d104cad5cb6e7df8b1775b40fcff44`.
The proxy allowed only public state hydration, no public exploit call or
transaction occurred, and the credential/path leak scan passed.
`NO_OUTCOME_ESTIMATES`.

## Exact inline files

Each marked block below is byte-identical to its attached file and contains
exactly one terminal LF. `poc/verify_inline.py` checks raw bytes without
trimming.

<!-- INLINE_FILE poc/foundry.toml -->
```toml
[profile.default]
src = "src"
out = "work/out"
cache_path = "work/cache"
solc = "0.8.35"
evm_version = "cancun"
optimizer = true
optimizer_runs = 200
via_ir = true
bytecode_hash = "ipfs"
```
<!-- END_INLINE_FILE poc/foundry.toml -->

<!-- INLINE_FILE poc/scenario.py -->
```python
#!/usr/bin/env python3
"""Standalone loopback-only PLETHER-AF-008 H378 control/attack target.

The target connects only to a pre-started local Anvil fork. It modifies no
in-scope runtime. Local code installation is restricted to external Pyth and
ArbSys compatibility adapters.
"""

from __future__ import annotations

import argparse
import json
from pathlib import Path

import evm as base


LANE = Path(__file__).resolve().parent
TARGET_IDENTITY = "AF008-H378-PORTABLE-R01"
LOCAL_RPC = "http://127.0.0.1:29175"
CHAIN_ID = 421_614
FORK_BLOCK = 326_291_776
FORK_BLOCK_HASH = "0xe3adca8a8fdc888cba9a3e7a41cb157b2580f21a21711168653eca584dd8399c"
FORK_TIMESTAMP = 1_793_880_481

BASELINE_MARK = 96_764_127
ATTACK_MARK = 97_731_768
VICTIM_REQUEST_TIMESTAMP = 1_793_880_570
EXPECTED_VICTIM_EPOCH_START = 1_793_883_600
PAIR_OPEN_TIMESTAMP = 1_793_887_200
ATTACK_TIMESTAMP = 1_794_001_200
T0_TIMESTAMP = 1_794_176_400
T1_TIMESTAMP = 1_794_262_801

VICTIM_DEPOSIT = 20_000_000 * 10**6
PAIR_SIZE = 67_800_000 * 10**18
LONG_FUNDING = 2_800_000 * 10**6
LONG_MARGIN = 2_200_000 * 10**6
SHORT_FUNDING = 1_200_000 * 10**6
SHORT_MARGIN = 650_000 * 10**6
CONTROLLED_CAPITAL = LONG_FUNDING + SHORT_FUNDING

EXPECTED_RISK = [10**16, 4 * 10**17, 10, 20, 300, 500, 10**6, 10, 5_000, 0]
EXPECTED_POOL_ASSETS = 102_206_129_390_293
EXPECTED_SIDE_MAX_PROFIT = [36_169_720_828_505, 234_263_191_564]
EXPECTED_SIDE_OPEN_INTEREST = [37_338_800 * 10**18, 227_000 * 10**18]
SIDE_LONG = 0
SIDE_SHORT = 1
NO_LP_EPOCH_PROGRESS_SELECTOR = 0x86CCA6B8
DEGRADED_SELECTOR = 0x9F7ADFF3

base.ANVIL_LOCAL_URL = LOCAL_RPC
base.FORK_BLOCK = FORK_BLOCK
base.CHAIN_ID = CHAIN_ID


def artifact(path: Path) -> dict:
    return json.loads(path.read_text(encoding="utf-8"))


def deploy_actor(args: argparse.Namespace, sender: str, label: str) -> str:
    build = artifact(args.lane / "work/out/H378CurrentConstructionActor.sol/H378CurrentConstructionActor.json")
    receipt = base.send(sender, None, base.object_hex(build["bytecode"]["object"]), label)
    address = receipt.get("contractAddress")
    if not address:
        raise AssertionError(f"{label}: missing contract address")
    return address.lower()


def install_external_adapters(args: argparse.Namespace) -> None:
    rows = (
        (
            base.PYTH,
            args.lane / "work/out/LocalPythH378AdapterV5.sol/LocalPythH378AdapterV5.json",
            "out-of-scope-pyth-adapter",
        ),
        (
            base.ARB_SYS,
            args.lane / "work/out/LocalArbSysAdapter.sol/LocalArbSysAdapter.json",
            "out-of-scope-arbsys-adapter",
        ),
    )
    for address, path, label in rows:
        runtime = base.object_hex(artifact(path)["deployedBytecode"]["object"])
        base.rpc("anvil_setCode", [address, runtime])
        base.assert_equal(
            label,
            base.keccak_hex(args, base.rpc("eth_getCode", [address, "latest"])),
            base.keccak_hex(args, runtime),
        )
    base.mine_empty_block("out-of-scope-arbsys-prime")
    before = base.uint_call(args, base.ARB_SYS, "arbBlockNumber()")
    base.mine_empty_block("out-of-scope-arbsys-advance")
    after = base.uint_call(args, base.ARB_SYS, "arbBlockNumber()")
    base.assert_gt("out-of-scope-arbsys-monotonic", after, before)


def side_state(args: argparse.Namespace, side: int) -> list[int]:
    values = base.words(base.call_raw(args, base.ENGINE, "sides(uint256)", side))
    if len(values) != 4:
        raise AssertionError(f"unexpected sides() width: {len(values)}")
    return values


def terminal_nav(args: argparse.Namespace) -> dict[str, int | bool]:
    values = base.words(base.call_raw(args, base.ENGINE, "terminalNavSnapshot()"))
    if len(values) != 8:
        raise AssertionError(f"unexpected terminalNavSnapshot() width: {len(values)}")
    return {
        "mark_price": values[0],
        "mark_time": values[1],
        "terminal_lp_price_delta_assets": base.signed(values[2]),
        "total_trader_claims_assets": values[3],
        "max_directional_liability_assets": values[4],
        "book_version": values[5],
        "has_open_positions": bool(values[6]),
        "degraded_mode": bool(values[7]),
    }


def preview_liquidation(args: argparse.Namespace, account: str, price: int) -> dict[str, int | bool]:
    values = base.words(base.call_raw(args, base.ENGINE_LENS, "previewLiquidation(address,uint256)", account, price))
    if len(values) != 20:
        raise AssertionError(f"unexpected liquidation preview width: {len(values)}")
    keys = (
        "liquidatable",
        "oracle_price",
        "equity_assets",
        "pnl_assets",
        "reachable_collateral_assets",
        "liquidation_charge_assets",
        "keeper_bounty_assets",
        "protocol_liquidation_fee_assets",
        "lp_liquidation_fee_assets",
        "seized_collateral_assets",
        "settlement_retained_assets",
        "fresh_trader_payout_assets",
        "existing_trader_claim_consumed_assets",
        "existing_trader_claim_remaining_assets",
        "immediate_payout_assets",
        "trader_claim_balance_assets",
        "triggers_degraded_mode",
        "post_op_degraded_mode",
        "effective_assets_after_assets",
        "max_liability_after_assets",
    )
    result = dict(zip(keys, values, strict=True))
    for key in ("liquidatable", "triggers_degraded_mode", "post_op_degraded_mode"):
        result[key] = bool(result[key])
    result["equity_assets"] = base.signed(int(result["equity_assets"]))
    result["pnl_assets"] = base.signed(int(result["pnl_assets"]))
    return result


def actor_selector(args: argparse.Namespace, actor: str) -> int:
    raw = base.call_raw(args, actor, "lastRevertSelector()")
    if len(raw) != 66:
        raise AssertionError("invalid bytes4 ABI response")
    return int(raw[2:10], 16)


def expected_selector(args: argparse.Namespace, signature: str) -> int:
    return int(base.cast(args, "sig", signature), 16)


def configuration_check(args: argparse.Namespace) -> dict[str, object]:
    base.assert_equal("release-risk-params", base.words(base.call_raw(args, base.ENGINE, "riskParams()")), EXPECTED_RISK)
    base.assert_equal("release-execution-fee-bps", base.uint_call(args, base.ENGINE, "executionFeeBps()"), 4)
    base.assert_equal("release-settlement-buffer-bps", base.uint_call(args, base.ENGINE, "settlementBufferBps()"), 25)
    base.assert_equal("release-trading-active", base.uint_call(args, base.POOL, "isTradingActive()"), 1)
    base.assert_equal("release-pool-unpaused", base.uint_call(args, base.POOL, "paused()"), 0)
    base.assert_equal("release-lp-settlement-unpaused", base.uint_call(args, base.POOL, "lpEpochSettlementPaused()"), 0)
    base.assert_equal("release-engine-not-degraded", base.uint_call(args, base.ENGINE, "degradedMode()"), 0)
    base.assert_equal("initial-pool-assets", base.uint_call(args, base.POOL, "totalAssets()"), EXPECTED_POOL_ASSETS)
    base.assert_equal("initial-total-claims", base.uint_call(args, base.ENGINE, "totalTraderClaimBalanceAssets()"), 0)
    sides = [side_state(args, side) for side in (SIDE_LONG, SIDE_SHORT)]
    base.assert_equal("initial-side-max-profit", [row[0] for row in sides], EXPECTED_SIDE_MAX_PROFIT)
    base.assert_equal("initial-side-open-interest", [row[1] for row in sides], EXPECTED_SIDE_OPEN_INTEREST)
    base.assert_equal("initial-order-queue-head", base.uint_call(args, base.ROUTER, "nextExecuteId()"), 0)
    state = {
        "pool_assets": EXPECTED_POOL_ASSETS,
        "sides": sides,
        "last_mark_price": base.uint_call(args, base.ENGINE, "lastMarkPrice()"),
        "last_mark_time": base.uint_call(args, base.ENGINE, "lastMarkTime()"),
    }
    base.record("exact-current-configuration", "MEASURED", **state)
    return state


def validate_oracle(args: argparse.Namespace, marker: int, publish_time: int, label: str) -> dict[str, object]:
    if label == "pair-open-oracle":
        result: dict[str, object] = {
            "mark_price": base.uint_call(args, base.ENGINE, "lastMarkPrice()"),
            "mark_time": base.uint_call(args, base.ENGINE, "lastMarkTime()"),
            "observation_domain": "historical-order-engine-mark",
        }
        base.assert_equal(f"{label}-engine-mark", result["mark_price"], marker)
        base.assert_equal(f"{label}-engine-time", result["mark_time"], publish_time)
        base.record(label, "MEASURED", **result)
        return result

    values = base.words(base.call_raw(args, base.PLETHER_ORACLE, "getLatestPoolReconcilePrice()"))
    if len(values) != 9:
        raise AssertionError(f"{label}: unexpected oracle return width {len(values)}")
    result = {
        "price": values[0],
        "mark_price": values[1],
        "publish_time": values[2],
        "confidence": values[8],
    }
    base.assert_equal(f"{label}-price", result["price"], marker)
    base.assert_equal(f"{label}-mark-price", result["mark_price"], marker)
    base.assert_equal(f"{label}-publish-time", result["publish_time"], publish_time)
    base.assert_equal(f"{label}-confidence", result["confidence"], 0)
    base.assert_equal(f"{label}-engine-mark", base.uint_call(args, base.ENGINE, "lastMarkPrice()"), marker)
    base.record(label, "MEASURED", **result)
    return result

def refresh_mark(
    args: argparse.Namespace,
    sender: str,
    keeper: str,
    marker: int,
    timestamp: int,
    label: str,
) -> dict[str, int]:
    base.set_next_timestamp(timestamp)
    base.actor_send(
        args,
        sender,
        keeper,
        "updateMarkPrice(uint256,uint64)",
        (marker, timestamp),
        label,
    )
    return validate_oracle(args, marker, timestamp, f"{label}-oracle")


def victim_state(args: argparse.Namespace, victim: str, request_id: int) -> dict[str, int]:
    return {
        "request_id": request_id,
        "pending_assets": base.uint_call(
            args, base.JUNIOR, "pendingDepositRequest(uint256,address)", request_id, victim
        ),
        "claimable_assets": base.uint_call(
            args, base.JUNIOR, "claimableDepositRequest(uint256,address)", request_id, victim
        ),
        "refundable_assets": base.uint_call(
            args, base.JUNIOR, "refundableDepositRequest(uint256,address)", request_id, victim
        ),
        "vault_token_balance": base.token_balance(args, base.JUNIOR),
    }


def assert_pending_victim(args: argparse.Namespace, victim: str, request_id: int, label: str) -> dict[str, int]:
    state = victim_state(args, victim, request_id)
    base.assert_equal(f"{label}-pending", state["pending_assets"], VICTIM_DEPOSIT)
    base.assert_equal(f"{label}-claimable", state["claimable_assets"], 0)
    base.assert_equal(f"{label}-refundable", state["refundable_assets"], 0)
    base.record(label, "MEASURED", **state)
    return state


def assert_claimable_victim(args: argparse.Namespace, victim: str, request_id: int, label: str) -> dict[str, int]:
    state = victim_state(args, victim, request_id)
    base.assert_equal(f"{label}-pending", state["pending_assets"], 0)
    base.assert_equal(f"{label}-claimable", state["claimable_assets"], VICTIM_DEPOSIT)
    base.assert_equal(f"{label}-refundable", state["refundable_assets"], 0)
    base.record(label, "MEASURED", **state)
    return state


def actor_state(args: argparse.Namespace, actor: str) -> dict[str, object]:
    return {
        "position": base.position_words(args, actor),
        "token_assets": base.token_balance(args, actor),
        "clearing_assets": base.clearing_balance(args, actor),
        "claim_assets": base.uint_call(args, base.ENGINE, "traderClaimBalanceAssets(address)", actor),
        "locked_margin_assets": base.uint_call(args, base.CLEARINGHOUSE, "lockedMarginAssets(address)", actor),
        "liquidation_reserve_assets": base.uint_call(
            args, base.CLEARINGHOUSE, "liquidationReserveAssets(address)", actor
        ),
        "bounty_reservations_assets": base.uint_call(
            args, base.CLEARINGHOUSE, "totalBountyReservationsAssets(address)", actor
        ),
        "pending_orders": base.uint_call(args, base.ROUTER, "pendingOrderCounts(address)", actor),
        "unsettled_carry_assets": base.uint_call(args, base.ENGINE, "unsettledCarryAssets(address)", actor),
    }


def assert_actor_flat(args: argparse.Namespace, actor: str, label: str) -> dict[str, object]:
    state = actor_state(args, actor)
    base.assert_equal(f"{label}-position", state["position"][0], 0)
    base.assert_equal(f"{label}-claim", state["claim_assets"], 0)
    base.assert_equal(f"{label}-locked-margin", state["locked_margin_assets"], 0)
    base.assert_equal(f"{label}-liquidation-reserve", state["liquidation_reserve_assets"], 0)
    base.assert_equal(f"{label}-bounty-reservations", state["bounty_reservations_assets"], 0)
    base.assert_equal(f"{label}-pending-orders", state["pending_orders"], 0)
    base.assert_equal(f"{label}-unsettled-carry", state["unsettled_carry_assets"], 0)
    return state


def setup(args: argparse.Namespace, accounts: list[str]) -> tuple[dict[str, str], dict[str, str], int, dict]:
    if len(accounts) < 4:
        raise RuntimeError("four distinct local development accounts required")
    senders = {
        role: accounts[index].lower()
        for index, role in enumerate(("victim", "long", "short", "keeper"))
    }
    base.assert_equal("distinct-role-controllers", len(set(senders.values())), 4)
    actors = {role: deploy_actor(args, senders[role], f"deploy-{role}") for role in senders}
    for role, actor in actors.items():
        base.assert_equal(f"{role}-controller", base.address_call(args, actor, "controller()"), senders[role])

    install_external_adapters(args)
    refresh_mark(args, senders["keeper"], actors["keeper"], BASELINE_MARK, 1_793_880_520, "baseline-mark")

    base.actor_send(args, senders["victim"], actors["victim"], "mintSelf(uint256)", (VICTIM_DEPOSIT,), "victim-mint")
    base.set_next_timestamp(VICTIM_REQUEST_TIMESTAMP)
    base.actor_send(
        args,
        senders["victim"],
        actors["victim"],
        "requestJuniorDepositExisting(uint256)",
        (VICTIM_DEPOSIT,),
        "victim-request-deposit",
    )
    request_id = base.actor_uint(args, actors["victim"], "lastDepositRequestId()")
    epoch_start = base.uint_call(args, base.JUNIOR, "depositEpochStart(uint256)", request_id)
    base.assert_equal("victim-epoch-start", epoch_start, EXPECTED_VICTIM_EPOCH_START)

    base.set_next_timestamp(epoch_start + 1)
    base.actor_send(
        args,
        senders["victim"],
        actors["victim"],
        "tryCancelJuniorDeposit(uint256)",
        (request_id,),
        "victim-mature-cancel-attempt",
    )
    base.assert_equal("victim-mature-cancel-ok", base.actor_uint(args, actors["victim"], "lastCallOk()"), 0)
    base.assert_equal(
        "victim-mature-cancel-selector",
        actor_selector(args, actors["victim"]),
        expected_selector(args, "TrancheVault__DepositEpochAlreadyActive()"),
    )
    assert_pending_victim(args, actors["victim"], request_id, "mature-victim-before-pair")

    for role, funding in (("long", LONG_FUNDING), ("short", SHORT_FUNDING)):
        base.actor_send(args, senders[role], actors[role], "mintSelf(uint256)", (funding,), f"{role}-mint")
        base.actor_send(
            args,
            senders[role],
            actors[role],
            "depositExisting(uint256)",
            (funding,),
            f"{role}-deposit",
        )

    base.set_next_timestamp(PAIR_OPEN_TIMESTAMP - 10)
    base.actor_send(
        args,
        senders["short"],
        actors["short"],
        "commitOpen(uint8,uint256,uint256)",
        (SIDE_SHORT, PAIR_SIZE, SHORT_MARGIN),
        "short-open-commit",
    )
    short_order_id = base.actor_uint(args, actors["short"], "lastOrderId()")
    base.set_next_timestamp(PAIR_OPEN_TIMESTAMP - 8)
    base.actor_send(
        args,
        senders["long"],
        actors["long"],
        "commitOpen(uint8,uint256,uint256)",
        (SIDE_LONG, PAIR_SIZE, LONG_MARGIN),
        "long-open-commit",
    )
    long_order_id = base.actor_uint(args, actors["long"], "lastOrderId()")
    base.assert_equal("pair-open-consecutive-order-ids", long_order_id, short_order_id + 1)
    base.set_next_timestamp(PAIR_OPEN_TIMESTAMP)
    base.actor_send(
        args,
        senders["keeper"],
        actors["keeper"],
        "executeOrderBatch(uint64,uint256,uint64)",
        (long_order_id, BASELINE_MARK, PAIR_OPEN_TIMESTAMP),
        "pair-open-batch",
    )
    base.assert_equal("pair-open-terminal-count", base.actor_uint(args, actors["keeper"], "lastOrderBatchTerminalCount()"), 2)
    base.assert_equal("pair-open-stop-reason", base.actor_uint(args, actors["keeper"], "lastOrderBatchStopReason()"), 0)
    base.assert_equal("pair-open-next-order", base.actor_uint(args, actors["keeper"], "lastOrderBatchNextId()"), 0)
    base.assert_equal("pair-open-queue-head", base.uint_call(args, base.ROUTER, "nextExecuteId()"), 0)
    base.assert_equal("pair-open-long-size", base.position_words(args, actors["long"])[0], PAIR_SIZE)
    base.assert_equal("pair-open-short-size", base.position_words(args, actors["short"])[0], PAIR_SIZE)
    base.assert_equal("pair-open-engine-healthy", base.uint_call(args, base.ENGINE, "degradedMode()"), 0)
    validate_oracle(args, BASELINE_MARK, PAIR_OPEN_TIMESTAMP, "pair-open-oracle")
    assert_pending_victim(args, actors["victim"], request_id, "mature-victim-after-pair")

    shared = {
        "senders": senders,
        "actors": actors,
        "request_id": request_id,
        "pair_positions": {
            "long": base.position_words(args, actors["long"]),
            "short": base.position_words(args, actors["short"]),
        },
        "pool_assets": base.uint_call(args, base.POOL, "totalAssets()"),
        "sides": [side_state(args, side) for side in (SIDE_LONG, SIDE_SHORT)],
        "victim": victim_state(args, actors["victim"], request_id),
    }
    base.record("shared-pre-liquidation-state", "MEASURED", **shared)
    return senders, actors, request_id, shared


def run_branch(
    args: argparse.Namespace,
    senders: dict[str, str],
    actors: dict[str, str],
    request_id: int,
) -> dict[str, object]:
    branch_gas_start = base._gas_used
    branch_fee_start = base._fee_wei
    branch_calldata_start = base._calldata_bytes

    refresh_mark(
        args,
        senders["keeper"],
        actors["keeper"],
        ATTACK_MARK,
        ATTACK_TIMESTAMP - 2,
        f"{args.mode}-attack-mark",
    )
    base.assert_equal(f"{args.mode}-fad-active", base.uint_call(args, base.ENGINE, "isFadWindow()"), 1)
    base.assert_equal(f"{args.mode}-oracle-live", base.uint_call(args, base.ENGINE, "isOracleFrozen()"), 0)

    long_preview = preview_liquidation(args, actors["long"], ATTACK_MARK)
    short_preview = preview_liquidation(args, actors["short"], ATTACK_MARK)
    base.assert_equal(f"{args.mode}-long-liquidatable", long_preview["liquidatable"], True)
    base.assert_equal(f"{args.mode}-short-liquidatable", short_preview["liquidatable"], True)
    base.assert_equal(f"{args.mode}-long-does-not-trigger", long_preview["triggers_degraded_mode"], False)
    base.assert_equal(f"{args.mode}-short-triggers", short_preview["triggers_degraded_mode"], True)
    base.record(
        f"{args.mode}-pre-liquidation-previews",
        "MEASURED",
        long=long_preview,
        short=short_preview,
    )

    order = [actors["long"], actors["short"]] if args.mode == "control" else [actors["short"], actors["long"]]
    base.set_next_timestamp(ATTACK_TIMESTAMP)
    base.actor_send(
        args,
        senders["keeper"],
        actors["keeper"],
        "executeLiquidationBatch(address[],uint256,uint64)",
        (f"[{order[0]},{order[1]}]", ATTACK_MARK, ATTACK_TIMESTAMP),
        f"{args.mode}-liquidation-batch",
    )
    base.assert_equal(
        f"{args.mode}-liquidation-next-index",
        base.actor_uint(args, actors["keeper"], "lastLiquidationBatchNextIndex()"),
        2,
    )
    long_flat = assert_actor_flat(args, actors["long"], f"{args.mode}-long-flat")
    short_flat = assert_actor_flat(args, actors["short"], f"{args.mode}-short-flat")
    expected_degraded = 0 if args.mode == "control" else 1
    base.assert_equal(
        f"{args.mode}-post-batch-degraded",
        base.uint_call(args, base.ENGINE, "degradedMode()"),
        expected_degraded,
    )
    post_sides = [side_state(args, side) for side in (SIDE_LONG, SIDE_SHORT)]
    base.assert_equal(f"{args.mode}-restored-side-max-profit", [row[0] for row in post_sides], EXPECTED_SIDE_MAX_PROFIT)
    base.assert_equal(
        f"{args.mode}-restored-side-open-interest",
        [row[1] for row in post_sides],
        EXPECTED_SIDE_OPEN_INTEREST,
    )
    nav = terminal_nav(args)
    pool_assets = base.uint_call(args, base.POOL, "totalAssets()")
    base.assert_gt(
        f"{args.mode}-raw-solvency-restored",
        pool_assets,
        int(nav["max_directional_liability_assets"]),
    )
    base.assert_equal(f"{args.mode}-nav-degraded", nav["degraded_mode"], bool(expected_degraded))
    assert_pending_victim(args, actors["victim"], request_id, f"{args.mode}-victim-post-batch")

    pre_withdraw_credits = {
        role: base.clearing_balance(args, actors[role]) for role in ("long", "short", "keeper")
    }
    for offset, role in enumerate(("long", "short", "keeper"), start=1):
        base.set_next_timestamp(ATTACK_TIMESTAMP + offset)
        base.actor_send(args, senders[role], actors[role], "withdrawAll()", (), f"{args.mode}-{role}-withdraw")
        base.assert_equal(f"{args.mode}-{role}-credit-zero", base.clearing_balance(args, actors[role]), 0)
    coalition_cash = sum(base.token_balance(args, actors[role]) for role in ("long", "short", "keeper"))
    net_cost = CONTROLLED_CAPITAL - coalition_cash
    ratio_wad = 0 if net_cost <= 0 else VICTIM_DEPOSIT * 10**18 // net_cost
    if net_cost > 0:
        base.assert_gt(f"{args.mode}-griefing-ratio-over-10x", ratio_wad, 10 * 10**18)
    economics = {
        "controlled_capital_assets": CONTROLLED_CAPITAL,
        "coalition_cash_assets": coalition_cash,
        "net_cost_assets": net_cost,
        "frozen_value_assets": VICTIM_DEPOSIT,
        "freeze_cost_ratio_wad": ratio_wad,
        "pre_withdraw_credits": pre_withdraw_credits,
        "token_cash": {
            role: base.token_balance(args, actors[role]) for role in ("long", "short", "keeper")
        },
    }
    base.record(f"{args.mode}-economics", "MEASURED", **economics)

    refresh_mark(
        args,
        senders["keeper"],
        actors["keeper"],
        ATTACK_MARK,
        T0_TIMESTAMP - 2,
        f"{args.mode}-t0-mark",
    )
    base.assert_equal(f"{args.mode}-t0-not-fad", base.uint_call(args, base.ENGINE, "isFadWindow()"), 0)
    base.assert_equal(f"{args.mode}-t0-oracle-live", base.uint_call(args, base.ENGINE, "isOracleFrozen()"), 0)
    base.set_next_timestamp(T0_TIMESTAMP)
    base.actor_send(
        args,
        senders["keeper"],
        actors["keeper"],
        "trySettleLpEpoch(uint256,uint64)",
        (ATTACK_MARK, T0_TIMESTAMP),
        f"{args.mode}-t0-settle",
    )
    t0_ok = bool(base.actor_uint(args, actors["keeper"], "lastCallOk()"))
    t0_selector = actor_selector(args, actors["keeper"])
    if args.mode == "control":
        base.assert_equal("control-t0-settle-ok", t0_ok, True)
        base.assert_equal("control-t0-selector", t0_selector, 0)
        t0_victim = assert_claimable_victim(args, actors["victim"], request_id, "control-t0-victim")
    else:
        base.assert_equal("attack-t0-settle-ok", t0_ok, False)
        base.assert_equal(
            "attack-t0-selector",
            t0_selector,
            expected_selector(args, "HousePool__DegradedMode()"),
        )
        t0_victim = assert_pending_victim(args, actors["victim"], request_id, "attack-t0-victim")

    refresh_mark(
        args,
        senders["keeper"],
        actors["keeper"],
        ATTACK_MARK,
        T1_TIMESTAMP - 2,
        f"{args.mode}-t1-mark",
    )
    base.assert_equal(f"{args.mode}-t1-not-fad", base.uint_call(args, base.ENGINE, "isFadWindow()"), 0)
    base.assert_equal(f"{args.mode}-t1-oracle-live", base.uint_call(args, base.ENGINE, "isOracleFrozen()"), 0)
    base.set_next_timestamp(T1_TIMESTAMP)
    base.actor_send(
        args,
        senders["keeper"],
        actors["keeper"],
        "trySettleLpEpoch(uint256,uint64)",
        (ATTACK_MARK, T1_TIMESTAMP),
        f"{args.mode}-t1-settle",
    )
    t1_ok = bool(base.actor_uint(args, actors["keeper"], "lastCallOk()"))
    t1_selector = actor_selector(args, actors["keeper"])
    if args.mode == "control":
        base.assert_equal("control-t1-settle-ok", t1_ok, False)
        base.assert_equal("control-t1-selector", t1_selector, NO_LP_EPOCH_PROGRESS_SELECTOR)
        t1_victim = assert_claimable_victim(args, actors["victim"], request_id, "control-t1-victim")
    else:
        base.assert_equal("attack-t1-settle-ok", t1_ok, False)
        base.assert_equal(
            "attack-t1-selector",
            t1_selector,
            expected_selector(args, "HousePool__DegradedMode()"),
        )
        t1_victim = assert_pending_victim(args, actors["victim"], request_id, "attack-t1-victim")
        base.assert_equal("attack-incremental-freeze-seconds", T1_TIMESTAMP - T0_TIMESTAMP, 24 * 60 * 60 + 1)

    base.manifest_runtime_check(args)
    return {
        "mode": args.mode,
        "result": "PASS",
        "liquidation_order": order,
        "previews": {"long": long_preview, "short": short_preview},
        "post_batch": {
            "long": long_flat,
            "short": short_flat,
            "sides": post_sides,
            "pool_assets": pool_assets,
            "terminal_nav": nav,
        },
        "economics": economics,
        "t0": {"ok": t0_ok, "selector": t0_selector, "victim": t0_victim},
        "t1": {"ok": t1_ok, "selector": t1_selector, "victim": t1_victim},
        "branch_gas_used": base._gas_used - branch_gas_start,
        "branch_receipt_fee_wei": base._fee_wei - branch_fee_start,
        "branch_calldata_bytes": base._calldata_bytes - branch_calldata_start,
    }


def run_target(args: argparse.Namespace, release: dict, accounts: list[str]) -> dict:
    initial = configuration_check(args)
    senders, actors, request_id, shared = setup(args, accounts)
    branch = run_branch(args, senders, actors, request_id)
    result = {
        "result": "PASS",
        "target_identity": TARGET_IDENTITY,
        "mode": args.mode,
        "chain_id": CHAIN_ID,
        "fork_block": FORK_BLOCK,
        "fork_block_hash": FORK_BLOCK_HASH,
        "fork_timestamp": FORK_TIMESTAMP,
        "source_commit": base.PINNED_COMMIT,
        "release_version": release["release"]["version"],
        "scenario": {
            "baseline_mark": BASELINE_MARK,
            "attack_mark": ATTACK_MARK,
            "victim_deposit_assets": VICTIM_DEPOSIT,
            "pair_size_each": PAIR_SIZE,
            "long_funding_assets": LONG_FUNDING,
            "long_margin_assets": LONG_MARGIN,
            "short_funding_assets": SHORT_FUNDING,
            "short_margin_assets": SHORT_MARGIN,
            "pair_open_timestamp": PAIR_OPEN_TIMESTAMP,
            "attack_timestamp": ATTACK_TIMESTAMP,
            "t0_timestamp": T0_TIMESTAMP,
            "t1_timestamp": T1_TIMESTAMP,
        },
        "initial_state": initial,
        "senders": senders,
        "actors": actors,
        "shared_pre_liquidation_state": shared,
        "branch": branch,
        "total_local_transactions": base._tx_count,
        "total_gas_used": base._gas_used,
        "total_receipt_fee_wei": base._fee_wei,
        "total_calldata_bytes": base._calldata_bytes,
        "public_transactions": 0,
        "public_node_eth_call": False,
        "owner_or_incumbent_impersonation": False,
        "snapshot_or_revert": False,
        "balance_or_storage_cheat": False,
        "in_scope_code_modified": False,
        "external_adapter_destinations": [base.PYTH, base.ARB_SYS],
    }
    base.record(f"h378-{args.mode}-result", "PASS", **result)
    return result


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--repo", type=Path, required=True)
    parser.add_argument("--foundry-bin", type=Path, required=True)
    parser.add_argument("--manifest", type=Path, required=True)
    parser.add_argument("--evidence-dir", type=Path, required=True)
    parser.add_argument("--expected-block", type=int, required=True)
    parser.add_argument("--expected-block-hash", required=True)
    parser.add_argument("--mode", choices=("control", "attack"), required=True)
    parser.add_argument("--ack-current-victim-poc", action="store_true")
    args = parser.parse_args()
    if not args.ack_current_victim_poc:
        parser.error("explicit local-fork acknowledgement is required")
    if args.expected_block != FORK_BLOCK or args.expected_block_hash.lower() != FORK_BLOCK_HASH:
        parser.error("frozen fork identity mismatch")
    args.lane = LANE
    args.repo = args.repo.resolve()
    args.foundry_bin = args.foundry_bin.resolve()
    args.cast = args.foundry_bin / "cast"
    args.manifest = args.manifest.resolve()
    args.evidence_dir = args.evidence_dir.resolve()
    if not args.evidence_dir.is_dir():
        parser.error("launcher-created evidence directory required")
    return args


def main() -> int:
    args = parse_args()
    base._run_handle = (args.evidence_dir / f"{args.mode}-run.jsonl").open("x", encoding="utf-8")
    result: dict[str, object] = {"result": "FAIL_PRESERVED", "mode": args.mode}
    try:
        base.assert_equal("local-chain-id", int(base.rpc("eth_chainId", []), 16), CHAIN_ID)
        block = base.rpc("eth_getBlockByNumber", ["latest", False])
        base.assert_equal("pinned-fork-block", int(block["number"], 16), FORK_BLOCK)
        base.assert_equal("pinned-fork-block-hash", block["hash"].lower(), FORK_BLOCK_HASH)
        base.assert_equal("pinned-fork-timestamp", int(block["timestamp"], 16), FORK_TIMESTAMP)
        release = base.manifest_runtime_check(args)
        base._logical_timestamp = FORK_TIMESTAMP
        result = run_target(args, release, base.rpc("eth_accounts", []))
    finally:
        with (args.evidence_dir / f"{args.mode}-result.json").open("x", encoding="utf-8") as handle:
            handle.write(json.dumps(result, indent=2, sort_keys=True) + "\n")
        if base._run_handle is not None:
            base._run_handle.close()
    if result.get("result") != "PASS":
        raise RuntimeError("H378 target failed; additive evidence preserved")
    print(f"H378 {args.mode.upper()} TARGET PASS")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```
<!-- END_INLINE_FILE poc/scenario.py -->

<!-- INLINE_FILE poc/evm.py -->
```python
#!/usr/bin/env python3
"""Generic loopback-only EVM helpers for the PLETHER-AF-008 local-fork proof.

This module has exactly one RPC endpoint: the literal local Anvil URL. It has no
upstream URL, credential, proxy startup, fork startup, or public-network path.
"""

from __future__ import annotations

import argparse
import hashlib
import json
import pathlib
import subprocess
import time
import urllib.error
import urllib.request


PINNED_COMMIT = "a82d4fefb02768e5b3bddcc97dee371eee24fea8"
FORK_BLOCK = 326_291_776
CHAIN_ID = 421_614
ANVIL_LOCAL_URL = "http://127.0.0.1:29175"

TOKEN = "0x3d051dbbf5ff427a4ae8ad09746902666af8fb9d"
CLEARINGHOUSE = "0xb5b851f627d6df3cf3f096e23e20a58884d7b89a"
ENGINE = "0xca18a6783d7542f9d2f28f4f259772b74c4d6baa"
ENGINE_LENS = "0xfca322c6868cef53bd5c344628f9874712e48108"
POOL = "0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e"
JUNIOR = "0x4a8b67184a1e3224c11a191cc69f22c50eef734e"
ROUTER = "0x918663d6da8cb33b325a04c4a5e8c51d42f3b444"
PYTH = "0x0b73614636c855bf23f342f307fb981a3e47f42b"
PLETHER_ORACLE = "0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2"
ARB_SYS = "0x0000000000000000000000000000000000000064"

_run_handle = None
_tx_count = 0
_gas_used = 0
_fee_wei = 0
_calldata_bytes = 0
_calldata_zero_bytes = 0
_calldata_nonzero_bytes = 0
_logical_timestamp = 0
_timestamp_armed = False


class RpcFailure(RuntimeError):
    pass


def output(*argv: str, cwd: pathlib.Path | None = None) -> str:
    return subprocess.check_output(list(argv), cwd=cwd, text=True).strip()


def record(step: str, status: str, **details) -> None:
    row = {"time": time.strftime("%Y-%m-%dT%H:%M:%S%z"), "step": step, "status": status, **details}
    if _run_handle is None:
        raise RuntimeError("run log is not open")
    _run_handle.write(json.dumps(row, sort_keys=True) + "\n")
    _run_handle.flush()
    print(f"{step}: {status}", flush=True)


def rpc(method: str, params: list):
    payload = json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": params}).encode()
    request = urllib.request.Request(ANVIL_LOCAL_URL, data=payload, headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(request, timeout=120) as response:
            body = json.loads(response.read())
    except urllib.error.HTTPError as error:
        raw = error.read()
        try:
            body = json.loads(raw)
        except json.JSONDecodeError as decode_error:
            raise RpcFailure(f"{method} HTTP {error.code} non-JSON response") from decode_error
    if "error" in body:
        error = body["error"]
        raise RpcFailure(
            f"{method}: code={error.get('code')} message={error.get('message')} data={error.get('data')}"
        )
    return body["result"]


def cast(args: argparse.Namespace, *values: str) -> str:
    return output(str(args.cast), *values, cwd=args.lane)


def calldata(args: argparse.Namespace, signature: str, *values) -> str:
    return cast(args, "calldata", signature, *(str(value) for value in values))


def keccak_hex(args: argparse.Namespace, value: str) -> str:
    return cast(args, "keccak", value).lower()


def artifact(args: argparse.Namespace, source_name: str, contract_name: str) -> dict:
    path = args.lane / "work/out" / source_name / f"{contract_name}.json"
    with path.open(encoding="utf-8") as handle:
        return json.load(handle)


def object_hex(value: str) -> str:
    return value if value.startswith("0x") else "0x" + value


def wait_receipt(tx_hash: str) -> dict:
    deadline = time.monotonic() + 120
    while time.monotonic() < deadline:
        receipt = rpc("eth_getTransactionReceipt", [tx_hash])
        if receipt is not None:
            return receipt
        time.sleep(0.05)
    raise RuntimeError(f"receipt timeout for {tx_hash}")


def send(sender: str, to: str | None, data: str, label: str) -> dict:
    global _tx_count, _gas_used, _fee_wei, _calldata_bytes, _calldata_zero_bytes
    global _calldata_nonzero_bytes, _timestamp_armed
    if _logical_timestamp == 0:
        raise RuntimeError("deterministic local clock is not initialized")
    if not _timestamp_armed:
        set_next_timestamp(_logical_timestamp + 1)
    expected_timestamp = _logical_timestamp
    transaction = {"from": sender, "data": data, "gas": hex(120_000_000), "value": "0x0"}
    if to is not None:
        transaction["to"] = to
    receipt = wait_receipt(rpc("eth_sendTransaction", [transaction]))
    block = rpc("eth_getBlockByNumber", [receipt["blockNumber"], False])
    block_timestamp = int(block["timestamp"], 16)
    _timestamp_armed = False
    status = int(receipt["status"], 16)
    gas_used = int(receipt["gasUsed"], 16)
    gas_price = int(receipt.get("effectiveGasPrice", "0x0"), 16)
    if not data.startswith("0x") or len(data[2:]) % 2:
        raise AssertionError(f"{label}: malformed transaction data")
    raw_data = bytes.fromhex(data[2:])
    zero_bytes = raw_data.count(0)
    nonzero_bytes = len(raw_data) - zero_bytes
    _tx_count += 1
    _gas_used += gas_used
    _fee_wei += gas_used * gas_price
    _calldata_bytes += len(raw_data)
    _calldata_zero_bytes += zero_bytes
    _calldata_nonzero_bytes += nonzero_bytes
    details = {
        "sender": sender,
        "destination": to,
        "transaction_hash": receipt["transactionHash"],
        "block_number": int(receipt["blockNumber"], 16),
        "block_timestamp": block_timestamp,
        "expected_block_timestamp": expected_timestamp,
        "gas_used": gas_used,
        "effective_gas_price_wei": gas_price,
        "receipt_fee_wei": gas_used * gas_price,
        "calldata_bytes": len(raw_data),
        "calldata_zero_bytes": zero_bytes,
        "calldata_nonzero_bytes": nonzero_bytes,
    }
    timestamp_matches = block_timestamp == expected_timestamp
    record(label, "PASS" if status == 1 and timestamp_matches else "FAIL", **details)
    if not timestamp_matches:
        raise AssertionError(
            f"{label}: block timestamp {block_timestamp} does not equal assigned {expected_timestamp}"
        )
    if status != 1:
        try:
            diagnostic_result = rpc("eth_call", [transaction, "latest"])
            record(
                f"{label}-local-revert-diagnostic",
                "UNEXPECTED_SUCCESS",
                result=diagnostic_result,
            )
        except RpcFailure as error:
            record(
                f"{label}-local-revert-diagnostic",
                "MEASURED",
                rpc_error=str(error),
            )
        raise AssertionError(f"{label}: local transaction reverted")
    return receipt


def call_raw(args: argparse.Namespace, to: str, signature: str, *values) -> str:
    return rpc("eth_call", [{"to": to, "data": calldata(args, signature, *values)}, "latest"])


def words(raw: str) -> list[int]:
    payload = raw[2:]
    if len(payload) % 64:
        raise AssertionError("non-word-aligned ABI response")
    return [int(payload[index : index + 64], 16) for index in range(0, len(payload), 64)]


def signed(value: int) -> int:
    return value - 2**256 if value >= 2**255 else value


def uint_call(args: argparse.Namespace, to: str, signature: str, *values) -> int:
    result = words(call_raw(args, to, signature, *values))
    if len(result) != 1:
        raise AssertionError(f"{signature}: expected one word, got {len(result)}")
    return result[0]


def address_call(args: argparse.Namespace, to: str, signature: str, *values) -> str:
    raw = call_raw(args, to, signature, *values)
    if len(raw) != 66:
        raise AssertionError(f"{signature}: invalid address response")
    return "0x" + raw[-40:].lower()


def assert_equal(label: str, observed, expected) -> None:
    ok = observed == expected
    record(label, "PASS" if ok else "FAIL", observed=observed, expected=expected)
    if not ok:
        raise AssertionError(f"{label}: observed {observed!r}, expected {expected!r}")


def assert_gt(label: str, observed: int, boundary: int) -> None:
    ok = observed > boundary
    record(label, "PASS" if ok else "FAIL", observed=observed, boundary=boundary)
    if not ok:
        raise AssertionError(f"{label}: {observed} is not greater than {boundary}")


def assert_le(label: str, observed: int, boundary: int) -> None:
    ok = observed <= boundary
    record(label, "PASS" if ok else "FAIL", observed=observed, boundary=boundary)
    if not ok:
        raise AssertionError(f"{label}: {observed} is not at or below {boundary}")


def current_timestamp() -> int:
    return int(rpc("eth_getBlockByNumber", ["latest", False])["timestamp"], 16)


def set_next_timestamp(timestamp: int) -> None:
    global _logical_timestamp, _timestamp_armed
    if timestamp <= current_timestamp():
        raise AssertionError("next timestamp must advance")
    rpc("evm_setNextBlockTimestamp", [timestamp])
    _logical_timestamp = timestamp
    _timestamp_armed = True


def mine_empty_block(label: str) -> None:
    global _timestamp_armed
    if _logical_timestamp == 0:
        raise RuntimeError("deterministic local clock is not initialized")
    if not _timestamp_armed:
        set_next_timestamp(_logical_timestamp + 1)
    expected_timestamp = _logical_timestamp
    rpc("evm_mine", [])
    observed_timestamp = current_timestamp()
    _timestamp_armed = False
    assert_equal(f"{label}-timestamp", observed_timestamp, expected_timestamp)


def actor_send(args: argparse.Namespace, sender: str, actor: str, signature: str, values: tuple, label: str) -> None:
    send(sender, actor, calldata(args, signature, *values), label)


def actor_uint(args: argparse.Namespace, actor: str, signature: str) -> int:
    return uint_call(args, actor, signature)


def token_balance(args: argparse.Namespace, account: str) -> int:
    return uint_call(args, TOKEN, "balanceOf(address)", account)


def clearing_balance(args: argparse.Namespace, account: str) -> int:
    return uint_call(args, CLEARINGHOUSE, "balanceAssets(address)", account)


def position_words(args: argparse.Namespace, account: str) -> list[int]:
    values = words(call_raw(args, ENGINE, "positions(address)", account))
    if len(values) != 7:
        raise AssertionError(f"unexpected positions() width: {len(values)}")
    return values


def manifest_runtime_check(args: argparse.Namespace) -> dict:
    with args.manifest.open(encoding="utf-8") as handle:
        release = json.load(handle)
    rows = []
    for name, item in release["contracts"].items():
        address = item["address"].lower()
        actual = keccak_hex(args, rpc("eth_getCode", [address, "latest"]))
        expected = item["runtimeCodeHash"].lower()
        rows.append({"name": name, "address": address, "actual": actual, "expected": expected})
        if actual != expected:
            record("runtime-hashes", "FAIL", rows=rows)
            raise AssertionError(f"runtime mismatch: {name}")
    assert_equal("runtime-count", len(rows), 30)
    record("runtime-hashes", "PASS", count=len(rows), rows=rows)
    return release
```
<!-- END_INLINE_FILE poc/evm.py -->

<!-- INLINE_FILE poc/compare.py -->
```python
#!/usr/bin/env python3
"""Cross-compare the two fresh-fork PLETHER-AF-008 result files."""

import argparse
import json
from pathlib import Path


def eq(label, observed, expected):
    if observed != expected:
        raise AssertionError(f"{label}: {observed!r} != {expected!r}")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("evidence", type=Path)
    args = parser.parse_args()
    control = json.loads((args.evidence / "control-result.json").read_text())
    attack = json.loads((args.evidence / "attack-result.json").read_text())
    for key in ("scenario", "initial_state", "senders", "actors", "shared_pre_liquidation_state"):
        eq(key, control[key], attack[key])
    cb, ab = control["branch"], attack["branch"]
    eq("previews", cb["previews"], ab["previews"])
    eq("control order", cb["liquidation_order"], [control["actors"]["long"], control["actors"]["short"]])
    eq("attack order", ab["liquidation_order"], [attack["actors"]["short"], attack["actors"]["long"]])
    eq("control degraded", cb["post_batch"]["terminal_nav"]["degraded_mode"], False)
    eq("attack degraded", ab["post_batch"]["terminal_nav"]["degraded_mode"], True)
    cn, an = dict(cb["post_batch"]["terminal_nav"]), dict(ab["post_batch"]["terminal_nav"])
    cn.pop("degraded_mode")
    an.pop("degraded_mode")
    eq("terminal NAV except bit", cn, an)
    eq("pool equality", cb["post_batch"]["pool_assets"], ab["post_batch"]["pool_assets"])
    eq("terminal pool assets", cb["post_batch"]["pool_assets"], 102_311_266_186_296)
    eq("incumbent open book", cn["has_open_positions"], True)
    eq("incumbent max liability", cn["max_directional_liability_assets"], 36_169_720_828_505)
    eq("terminal trader claims", cn["total_trader_claims_assets"], 0)
    eq("terminal LP-price delta", cn["terminal_lp_price_delta_assets"], 319_658_095_248)
    eq("branch economics", cb["economics"], ab["economics"])
    economics = ab["economics"]
    eq("controlled capital", economics["controlled_capital_assets"], 4_000_000_000_000)
    eq("coalition cash", economics["coalition_cash_assets"], 3_842_378_341_513)
    eq("pre-gas cost", economics["net_cost_assets"], 157_621_658_487)
    eq("frozen value", economics["frozen_value_assets"], 20_000_000_000_000)
    eq("ratio WAD", economics["freeze_cost_ratio_wad"], 126_886_115_727_868_194_614)
    eq("cash components", economics["token_cash"],
       {"keeper": 65_606_478_106, "long": 2_021_107_986_342, "short": 1_755_663_877_065})
    for mode, branch in (("control", cb), ("attack", ab)):
        for role in ("long", "short"):
            state = branch["post_batch"][role]
            eq(f"{mode} {role} position", state["position"], [0] * 7)
            for field in ("claim_assets", "locked_margin_assets", "liquidation_reserve_assets",
                          "bounty_reservations_assets", "pending_orders", "unsettled_carry_assets"):
                eq(f"{mode} {role} {field}", state[field], 0)
    eq("control T0", (cb["t0"]["ok"], cb["t0"]["selector"]), (True, 0))
    eq("control T1", (cb["t1"]["ok"], cb["t1"]["selector"]), (False, 0x86CCA6B8))
    eq("control claimable", cb["t1"]["victim"]["claimable_assets"], 20_000_000_000_000)
    for checkpoint in ("t0", "t1"):
        row = ab[checkpoint]
        eq(f"attack {checkpoint}", (row["ok"], row["selector"]), (False, 0x9F7ADFF3))
        eq(f"attack {checkpoint} victim",
           (row["victim"]["pending_assets"], row["victim"]["claimable_assets"], row["victim"]["refundable_assets"]),
           (20_000_000_000_000, 0, 0))
    eq("freeze seconds", attack["scenario"]["t1_timestamp"] - attack["scenario"]["t0_timestamp"], 86_401)
    print("AF008_H378_DIFFERENTIAL_PASS")
    print("CONTROL_ORDER LONG_SHORT DEGRADED 0 CLAIMABLE 20000000000000")
    print("ATTACK_ORDER SHORT_LONG DEGRADED 1 PENDING 20000000000000")
    print("FREEZE_SECONDS 86401 PRE_GAS_COST 157621658487 RATIO_WAD 126886115727868194614")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```
<!-- END_INLINE_FILE poc/compare.py -->

<!-- INLINE_FILE poc/src/H378CurrentConstructionActor.sol -->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

interface IH378Token {

    function mint(
        address account,
        uint256 amount
    ) external;

    function approve(
        address spender,
        uint256 amount
    ) external returns (bool);

    function balanceOf(
        address account
    ) external view returns (uint256);

}

interface IH378Clearinghouse {

    function deposit(
        address account,
        uint256 amount
    ) external;

    function withdraw(
        address account,
        uint256 amount
    ) external;

    function balanceAssets(
        address account
    ) external view returns (uint256);

}

interface IH378Router {

    function commitOrder(
        OrderV3Types.OrderRequest calldata request
    ) external returns (uint64 orderId);

    function executeOrderBatch(
        uint64 maxOrderId,
        bytes[] calldata updateData
    ) external payable returns (OrderV3Types.BatchResult memory result);

    function executeLiquidationBatch(
        address[] calldata accounts,
        bytes[] calldata updateData
    ) external payable returns (uint256 nextIndex);

    function updateMarkPrice(
        bytes[] calldata updateData
    ) external payable;

    function settleLpEpoch(
        bytes[] calldata updateData
    ) external payable;

    function maxExecutionWindowSeconds() external view returns (uint256);

    function lifecycleBook() external view returns (IOrderLifecycleBook);

}

interface IH378JuniorVault {

    function requestDeposit(
        uint256 assets,
        address controller,
        address owner
    ) external returns (uint256 requestId);

    function cancelPendingDeposit(
        uint256 requestId
    ) external returns (uint256 assets);

}

/// @notice Ordinary-account helper for the exact-current H378 local-fork control/attack proof.
/// @dev It has no admin method, arbitrary-call primitive, impersonation, or target-mutation capability.
contract H378CurrentConstructionActor {

    address internal constant TOKEN = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    uint256 internal constant CAP_PRICE = 200_000_000;

    address private immutable CONTROLLER;
    uint256 public nonce;
    uint64 public lastOrderId;
    uint64 public lastOrderBatchNextId;
    uint32 public lastOrderBatchTerminalCount;
    OrderV3Types.PendingReason public lastOrderBatchStopReason;
    uint256 public lastLiquidationBatchNextIndex;
    uint256 public lastDepositRequestId;
    bool public lastCallOk;
    bytes4 public lastRevertSelector;

    constructor() {
        CONTROLLER = msg.sender;
    }

    modifier onlyController() {
        require(msg.sender == CONTROLLER, "controller");
        _;
    }

    function controller() external view returns (address) {
        return CONTROLLER;
    }

    function mintSelf(
        uint256 amount
    ) external onlyController {
        IH378Token(TOKEN).mint(address(this), amount);
    }

    function depositExisting(
        uint256 amount
    ) external onlyController {
        require(IH378Token(TOKEN).approve(CLEARINGHOUSE, amount), "clearinghouse approval");
        IH378Clearinghouse(CLEARINGHOUSE).deposit(address(this), amount);
    }

    function requestJuniorDepositExisting(
        uint256 amount
    ) external onlyController returns (uint256 requestId) {
        require(IH378Token(TOKEN).approve(JUNIOR, amount), "junior approval");
        requestId = IH378JuniorVault(JUNIOR).requestDeposit(amount, address(this), address(this));
        lastDepositRequestId = requestId;
    }

    function tryCancelJuniorDeposit(
        uint256 requestId
    ) external onlyController returns (bool ok, bytes4 selector) {
        bytes memory returnData;
        (ok, returnData) = JUNIOR.call(abi.encodeCall(IH378JuniorVault.cancelPendingDeposit, (requestId)));
        selector = _selector(returnData);
        lastCallOk = ok;
        lastRevertSelector = selector;
    }

    function commitOpen(
        uint8 sideRaw,
        uint256 size,
        uint256 margin
    ) external onlyController returns (uint64 orderId) {
        require(sideRaw <= uint8(CfdTypes.Side.SHORT), "side");
        CfdTypes.Side side = CfdTypes.Side(sideRaw);
        uint256 window = IH378Router(ROUTER).maxExecutionWindowSeconds();
        require(window <= type(uint32).max, "window overflow");
        require(block.timestamp <= type(uint64).max - window, "submit time overflow");
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode(address(this), ++nonce, side, size, margin));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : CAP_PRICE;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            // forge-lint: disable-next-line(unsafe-typecast)
            submitBy: uint64(block.timestamp + window),
            // forge-lint: disable-next-line(unsafe-typecast)
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: IH378Router(ROUTER).lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: size,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        orderId = IH378Router(ROUTER).commitOrder(request);
        lastOrderId = orderId;
    }

    function executeOrderBatch(
        uint64 maxOrderId,
        uint256 basketMarker,
        uint64 publishTime
    ) external onlyController {
        OrderV3Types.BatchResult memory result =
            IH378Router(ROUTER).executeOrderBatch(maxOrderId, _priceData(basketMarker, publishTime));
        lastOrderBatchNextId = result.nextOrderId;
        lastOrderBatchTerminalCount = result.terminalCount;
        lastOrderBatchStopReason = result.stopReason;
    }

    function executeLiquidationBatch(
        address[] calldata accounts,
        uint256 basketMarker,
        uint64 publishTime
    ) external onlyController {
        lastLiquidationBatchNextIndex =
            IH378Router(ROUTER).executeLiquidationBatch(accounts, _priceData(basketMarker, publishTime));
    }

    function updateMarkPrice(
        uint256 basketMarker,
        uint64 publishTime
    ) external onlyController {
        IH378Router(ROUTER).updateMarkPrice(_priceData(basketMarker, publishTime));
    }

    function trySettleLpEpoch(
        uint256 basketMarker,
        uint64 publishTime
    ) external onlyController returns (bool ok, bytes4 selector) {
        bytes memory returnData;
        (ok, returnData) =
            ROUTER.call(abi.encodeCall(IH378Router.settleLpEpoch, (_priceData(basketMarker, publishTime))));
        selector = _selector(returnData);
        lastCallOk = ok;
        lastRevertSelector = selector;
    }

    function withdrawAll() external onlyController returns (uint256 amount) {
        amount = IH378Clearinghouse(CLEARINGHOUSE).balanceAssets(address(this));
        if (amount != 0) {
            IH378Clearinghouse(CLEARINGHOUSE).withdraw(address(this), amount);
        }
    }

    function tokenBalance() external view returns (uint256) {
        return IH378Token(TOKEN).balanceOf(address(this));
    }

    function _priceData(
        uint256 basketMarker,
        uint64 publishTime
    ) private pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(basketMarker, publishTime);
    }

    function _selector(
        bytes memory returnData
    ) private pure returns (bytes4 selector) {
        if (returnData.length < 4) {
            return bytes4(0);
        }
        assembly ("memory-safe") {
            selector := mload(add(returnData, 0x20))
        }
    }

}
```
<!-- END_INLINE_FILE poc/src/H378CurrentConstructionActor.sol -->

<!-- INLINE_FILE poc/src/LocalPythH378AdapterV5.sol -->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";

/// @notice Local-only exact-current adapter for the baseline and calibrated +100 bps H378 marks.
/// @dev The vectors use production feed IDs and reproduce both registered basket marks exactly.
contract LocalPythH378AdapterV5 is IPyth {

    uint256 private constant BASELINE_MARKER = 96_764_127;
    uint256 private constant ATTACK_MARKER = 97_731_768;

    uint256 private latestMarker;
    uint64 private latestPublishTime;

    function getPriceUnsafe(
        bytes32 id
    ) external view returns (PythStructs.Price memory price) {
        return _storedPrice(id);
    }

    function getPriceNoOlderThan(
        bytes32 id,
        uint256 age
    ) external view returns (PythStructs.Price memory price) {
        price = _storedPrice(id);
        require(uint256(price.publishTime) + age >= block.timestamp, "stale local price");
    }

    function updatePriceFeeds(
        bytes[] calldata updateData
    ) external payable {
        (latestMarker, latestPublishTime) = _decode(updateData);
    }

    function parsePriceFeedUpdates(
        bytes[] calldata updateData,
        bytes32[] calldata priceIds,
        uint64 minPublishTime,
        uint64 maxPublishTime
    ) external payable returns (PythStructs.PriceFeed[] memory priceFeeds) {
        return _feeds(updateData, priceIds, minPublishTime, maxPublishTime);
    }

    function parsePriceFeedUpdatesUnique(
        bytes[] calldata updateData,
        bytes32[] calldata priceIds,
        uint64 minPublishTime,
        uint64 maxPublishTime
    ) external payable returns (PythStructs.PriceFeed[] memory priceFeeds) {
        return _feeds(updateData, priceIds, minPublishTime, maxPublishTime);
    }

    function getUpdateFee(
        bytes[] calldata
    ) external pure returns (uint256 feeAmount) {
        return 0;
    }

    function _storedPrice(
        bytes32 id
    ) private view returns (PythStructs.Price memory price) {
        require(latestPublishTime != 0, "local price not initialized");
        return _price(_indexOf(id), latestMarker, latestPublishTime);
    }

    function _feeds(
        bytes[] calldata updateData,
        bytes32[] calldata priceIds,
        uint64 minPublishTime,
        uint64 maxPublishTime
    ) private pure returns (PythStructs.PriceFeed[] memory priceFeeds) {
        (uint256 marker, uint64 publishTime) = _decode(updateData);
        if (publishTime < minPublishTime || publishTime > maxPublishTime) {
            revert IPyth.PriceFeedNotFoundWithinRange();
        }
        require(priceIds.length == 6, "unexpected feed count");
        priceFeeds = new PythStructs.PriceFeed[](priceIds.length);
        for (uint256 i; i < priceIds.length; ++i) {
            bytes32 expected = _feedId(i);
            require(priceIds[i] == expected, "unexpected feed order");
            PythStructs.Price memory value = _price(i, marker, publishTime);
            priceFeeds[i] = PythStructs.PriceFeed({id: expected, price: value, emaPrice: value});
        }
    }

    function _decode(
        bytes[] calldata updateData
    ) private pure returns (uint256 marker, uint64 publishTime) {
        require(updateData.length == 1, "one local price blob required");
        (marker, publishTime) = abi.decode(updateData[0], (uint256, uint64));
        require(marker == BASELINE_MARKER || marker == ATTACK_MARKER, "unregistered local fixture");
        require(publishTime != 0, "zero publish time");
    }

    function _price(
        uint256 index,
        uint256 marker,
        uint64 publishTime
    ) private pure returns (PythStructs.Price memory value) {
        return PythStructs.Price({price: _rawPrice(index, marker), conf: 0, expo: -8, publishTime: publishTime});
    }

    function _rawPrice(
        uint256 index,
        uint256 marker
    ) private pure returns (int64) {
        int64[6] memory baseline = [
            int64(113_697_860),
            int64(16_198_135_594),
            int64(130_128_397),
            int64(141_800_335),
            int64(951_602_994),
            int64(81_954_070)
        ];
        int64[6] memory attack = [
            int64(114_834_866),
            int64(16_037_758_013),
            int64(131_429_680),
            int64(140_396_371),
            int64(942_181_182),
            int64(81_142_643)
        ];
        require(index < baseline.length, "raw price index");
        return marker == BASELINE_MARKER ? baseline[index] : attack[index];
    }

    function _indexOf(
        bytes32 id
    ) private pure returns (uint256) {
        for (uint256 i; i < 6; ++i) {
            if (id == _feedId(i)) {
                return i;
            }
        }
        revert("unknown local feed");
    }

    function _feedId(
        uint256 index
    ) private pure returns (bytes32) {
        if (index == 0) {
            return 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        }
        if (index == 1) {
            return 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        }
        if (index == 2) {
            return 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        }
        if (index == 3) {
            return 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        }
        if (index == 4) {
            return 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        }
        if (index == 5) {
            return 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        }
        revert("feed index");
    }

}
```
<!-- END_INLINE_FILE poc/src/LocalPythH378AdapterV5.sol -->

<!-- INLINE_FILE poc/src/LocalArbSysAdapter.sol -->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

/// @notice Minimal local-Anvil compatibility adapter for Arbitrum's external ArbSys precompile.
contract LocalArbSysAdapter {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}
```
<!-- END_INLINE_FILE poc/src/LocalArbSysAdapter.sol -->

