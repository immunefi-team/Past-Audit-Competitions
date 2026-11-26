# \#98849 [SC-High] Atomic expire-and-retry lets any outsider consume the owner-only Latched repair window

**Submitted on Nov 10th 2026 at 14:57:10 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98849
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

Cecuro MED-9 reported that permissionless position-protection retries lacked
trader-selected financial ceilings. The frozen repair preserves four close
ceilings across attempts and lets only the position owner amend them while the
protection is `Latched`. Retry remains permissionless.

The two sides of that repair do not compose safely. A naturally expired close
attempt changes the protection from `Triggered` to `Latched`, but any caller
may immediately call `retryPositionProtectionClose`. An outsider contract can
therefore call `expireOrder(child)` and
`retryPositionProtectionClose(protection)` back-to-back in one transaction.
The first call creates the owner-only repair state; the second consumes it and
queues a new child before any owner transaction can be ordered between them.
The replacement keeps the retained ceilings and restores `Triggered`, so the
owner's documented amendment action reverts.

The attached local-fork proof demonstrates this composition against the exact
Arbitrum Sepolia v2.0.0 deployed comparator. The outsider has no owner,
guardian, governance, oracle, or victim privilege; supplies no false oracle
data; and starts and ends with zero settlement tokens. Its trigger is an
ordinary child reaching its configured deadline. A matched one-hunk control
that restricts retry to the owner rejects the identical outsider transaction
and rolls back all state, after which the owner can expire and retry normally.

This report uses the competition's explicit incomplete-fix exception: a report
on a fixed or Acknowledged audit root remains valid when the frozen fix is
incomplete or introduces a new bug. I am not claiming that permissionless
retry, retained limits, or the MED-9 family was previously unknown. The narrow
claim is that the repair introduced an owner-only `Latched` action without
making that state externally actionable before a foreign caller can consume
it atomically.

## Root cause

When a close attempt terminates for a permitted non-liquidation failure and
the exact protected position still exists,
`PositionProtectionBook.handleFailedProtectionAttempt` deletes the old child
mapping and performs:

```solidity
retained = _positionMatches(protection);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

The MED-9 repair action is owner-only through `_ownedActiveProtection` and is
available only in that state with no pending Router order:

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection =
        _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
}
```

In contrast, retry loads the record through `_activeProtection`, not
`_ownedActiveProtection`, and contains no caller check or priority interval:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);
    // ... no-pending and linked-child checks ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(
            protectionId, protection.account, protection.side, protection.size
        );
    _recordCloseAttempt(protection, linkedOrderId);
}
```

`expireOrder` is also public. Solidity transaction atomicity therefore lets a
foreign helper create `Latched` and consume it in the same transaction. The
ordinary cancellation and atomic cancel-and-close routes accept only
pre-trigger states; a `Triggered` or `Latched` protection remains binding.

## Executed trigger and impact

The proof binds source commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`, tree
`d88b5f9fca750cd69197b33e99a5189d3f1eccf2`, chain `421614`, and block
`325561334`. At target entry it checks all 30 v2.0.0 release runtime hashes.
After control installation it rechecks all 30, allowing exactly one disclosed
Book-runtime exception. Local code replacement in the vulnerable branch is
limited to Pyth and ArbSys adapters.

All numerical execution values in this section are **MEASURED** in the exact
clean-extracted r02 archive run. No intended-mainnet economic value or damage
ratio is derived from them.

In the vulnerable branch:

1. distinct victim, keeper, and outsider actors open a real protected position
   through the deployed stack;
2. child `33` naturally reaches its deadline and expiration returns protection
   `6` to `Latched`;
3. the owner successfully amends the retained ceilings from a
   `1,000,000`-atom notional ceiling to
   `[2,000,000; 25,000,000,000; 25,000,000,000; 25,000,000,000]`;
4. the outsider permissionlessly retries to child `34`;
5. after child `34` naturally reaches its deadline, one outsider transaction
   expires `34` and retries protection `6`, installing child `35`; and
6. the owner's immediate attempt to change all four ceilings to
   `25,000,000,000` atoms reverts with exact
   `OrderRouter__ProtectionNotLatched()` selector `0x5b54e624`.

The failed owner transaction consumes `33,103` local gas. The full candidate-
state SHA-256 is byte-for-byte equal before and after its revert:

```text
4354a60a6f9bd19756066d2353a71955f424168d05838a2e7381e34945f99338
```

At that point protection `6` is `Triggered`, child `35` is the sole pending
linked order and retains the prior `2,000,000`-atom notional ceiling. Position,
victim settlement balance, action reserve, and outsider settlement-token
balance are unchanged. The outsider starts and ends with zero settlement
tokens. The vulnerable branch uses 15 local transactions.

In the matched control, the only in-scope logic change is this check at the
start of retry:

```solidity
if (msg.sender != protection.account) {
    revert OrderRouter__Unauthorized();
}
```

The identical outsider expire-plus-retry call reverts with exact selector
`0xb02f6f95`; the complete state hash before and after is:

```text
6db3776d729d0fefb88b92f28710d327761c468e6b9cadcc7aea06c026c7e263
```

Child `34` remains the sole pending child. The owner then expires it and
successfully retries to child `35`. The control uses 17 local transactions.
This isolates permissionless consumption of `Latched` as the cause; owner-only
retry is diagnostic, not the only proposed remedy.

## Impact and severity

The directly demonstrated damage is denial of the owner's only documented
post-trigger ceiling-repair action. A successful outsider transaction renews
the pending protection attempt with the old limits, keeps the protection
binding, makes the immediate owner repair transaction revert, and costs the
outsider only native gas. The same structural opportunity returns when a later
child naturally reaches its deadline. The outsider must win ordering at each
renewal; the proof does not assume control of the oracle, keeper, owner, or
sequencer.

I select **Medium — Griefing (e.g. no profit motive for an attacker, but damage
to the users or the protocol)**. I do not claim attacker profit, principal
theft, permanent freezing, or a distinct victim-owned amount inaccessible for
one hour. A close attempt may execute, liquidation or position mutation may
terminalize the protection, a privileged containment action may intervene, or
the owner may win a later deadline race. Those recovery paths bound the claim
but do not remove the executed outsider-caused denial and gas damage.

## Preconditions and feasibility

- A protection has triggered and its current zero-bounty close attempt reaches
  its legitimate deadline while the exact protected position remains.
- The outsider monitors the public deadline and pays native gas for one public
  atomic transaction. No settlement-token capital is required.
- The outsider's transaction wins ordering over any competing owner/keeper
  action at that deadline.
- To renew the obstruction, each replacement child must again remain pending
  until its deadline and the outsider must win again.

These are natural/environmental conditions and public composition, not
malicious-oracle or trusted-role assumptions. The proof establishes exact
frozen-release mechanics, not a current intended-mainnet victim population,
real-USDG liquidity, production gas costs, or monetary value at risk.

## Knownness and strongest accepted-design objection

The overlap is material:

- Cecuro MED-9 disclosed permissionless retries without trader-selected
  ceilings. The acknowledged repair retained close limits and added the
  owner-only `Latched` amendment.
- SCAN-10 disclosed the early trigger-bounty/zero-bounty retry flow,
  noncancellable `Triggered`/`Latched` protection, and the resulting
  open-position loss/liquidation channel.
- Frozen documentation says retry is permissionless, says the owner may amend
  after a failure becomes `Latched`, and instructs keeper infrastructure to
  process the terminal attempt, observe `Latched`, then submit a separate retry
  transaction.
- The out-of-scope AA-client README says the owner can amend limits "before a
  permissionless retry." Public development history introduced retry races
  among keepers and later added amendment-before-retry coverage.

The strongest closure is therefore that the repair intentionally leaves a
public race and the quoted phrase already accepts that another caller may win.
That objection is substantial. The distinguishing evidence is that the
documented keeper sequence and test assume a separately observable `Latched`
state, whereas the public expiry and retry entrypoints can be composed by one
foreign helper so no owner transaction can occupy the interval. No current
public audit item, known-issue entry, source comment, documentation passage,
test, issue, or commit message found in the 10 November public refresh states
that exact atomic composition or says the owner-only repair state may be made
externally unreachable in this way.

Independently re-verified against the full public history, not merely the
current tree: `retryPositionProtectionClose` was introduced in
`Plether-Fi/plether-core` PR #78 ("feat(perps): add latched SL/TP close
retries") calling the unchecked `_activeProtection(protectionId)` from its
original introduction. PR #124 ("fix(perps): harden trader protection,
liquidation pricing, and manual exits") later added the sibling
`amendLatchedPositionProtectionCloseLimits` correctly owner-checked via
`_ownedActiveProtection(protectionId, msg.sender)`, but never touches
`retryPositionProtectionClose`'s own definition — the asymmetry has existed
unexamined across both PRs. The retained `PositionProtection.t.sol` calls
`retryPositionProtectionClose` from a non-owner address (`CAROL`) extensively
and as passing, intended behavior, but in every one of those call sites the
expiry and the retry are two separate statements (a keeper expires, then a
later, distinct `vm.prank`/call retries) — none of them compose the two into
one atomic transaction the way this proof's `expireAndRetry` helper does. No
test exercises the atomic composition itself. A full-history GitHub PR/issue
search for `PositionProtectionBook`, `amendLatchedPositionProtectionCloseLimits`,
and `retryPositionProtectionClose` across all states returned ten PRs and zero
issues; none discusses this atomic-composition race. The repository's own
`audits/scan-plether-fi-plether-core-d4541b49.md` static-scan references
`PositionProtectionBook.sol` extensively by line but never cites either of
these two functions or this authorization asymmetry.

This is deliberately presented as an **incomplete MED-9 repair**, not broad
novelty. Formal CVQC and the filing-time knownness check must still decide
whether the public wording makes the ordering behavior accepted design or the
same publicly disclosed root.

Cross-finding leakage census: the candidate fields and upload archive contain
only this atomic expiry-plus-retry mechanism, its diagnostic control, and the
MED-9/SCAN-10 material necessary to disclose public knownness. They contain no
trigger, exploit path, impact, or remediation for a separate unsubmitted
finding.

## Recommended remediation

Preserve permissionless liveness while guaranteeing that the owner-only repair
operation is usable. Possible designs include:

- an owner-priority block/time interval after relatching before a foreign
  retry;
- an owner-native atomic expire-and-amend operation;
- allowing the owner to stage limits that the next retry must consume even
  while the current attempt is `Triggered`; or
- requiring owner authorization for retry, as in the diagnostic control.

Add an integration regression in which a foreign contract atomically calls
expiry and retry at the natural deadline, and prove either that owner-staged
limits reach the replacement child or that the foreign retry cannot consume
the repair state before the defined priority boundary.

## References

- Competition scope, knownness rule, PoC rule, and impact table:
  <https://immunefi.com/audit-competition/plether/scope/>
- Competition timing and public-known-issue policy:
  <https://immunefi.com/audit-competition/plether/information/>
- Frozen source:
  <https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8>


## Link to Proof of Concept

https://gist.github.com/AFestive/77b261572c5a99f460fb8fc0d322f54f

## Proof of Concept

## Proof of Concept

Run per the attached README.md's `./run.sh` invocation. Vulnerable: child
35 installed, owner amend reverts `0x5b54e624`. Control: same call reverts
`0xb02f6f95`, rollback.

### Inline files (byte-identical)

<!--run_af011_local_target.py-->
```python
#!/usr/bin/env python3
"""One AF-011 R10 deterministic-time attempt against a loopback Anvil fork.

This target has no upstream endpoint, credential lookup, proxy orchestration, or
public-node path. It uses only ordinary unlocked local-Anvil EOAs and production
external functions after the launcher's pinned state hydration.
"""

from __future__ import annotations

import argparse
import datetime
import hashlib
import json
import os
import pathlib
import re
import subprocess
import time
import urllib.error
import urllib.request


PINNED_COMMIT = "a82d4fefb02768e5b3bddcc97dee371eee24fea8"
FORK_BLOCK = 325_561_334
CHAIN_ID = 421_614
LOCAL_URLS = {
    "vulnerable": "http://127.0.0.1:29811",
    "control": "http://127.0.0.1:29821",
}

TOKEN = "0x3d051dbbf5ff427a4ae8ad09746902666af8fb9d"
CLEARINGHOUSE = "0xb5b851f627d6df3cf3f096e23e20a58884d7b89a"
ENGINE = "0xca18a6783d7542f9d2f28f4f259772b74c4d6baa"
ROUTER = "0x918663d6da8cb33b325a04c4a5e8c51d42f3b444"
BOOK = "0x81e3b03cf29865b6c0f85925839567e2bc61073d"
PYTH = "0x0b73614636c855bf23f342f307fb981a3e47f42b"
ARB_SYS = "0x0000000000000000000000000000000000000064"

MARK_PRICE = 100_000_000
TRIGGER_PRICE = 90_000_000
POSITION_SIZE = 10_000 * 10**18
POSITION_MARGIN = 5_000 * 10**6
TRADER_FUNDS = 50_000 * 10**6
INITIAL_NOTIONAL_LIMIT = 1 * 10**6
AMENDED_NOTIONAL_LIMIT = 2 * 10**6
OTHER_LIMIT = 25_000 * 10**6
MAX_TX_GAS = 100_000_000
STATUS_ARMED = 2
STATUS_TRIGGERED = 3
STATUS_LATCHED = 8
LIFECYCLE_PENDING = 1
PROTECTION_BOUNTY_KIND = 1

FEED_IDS = (
    "0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b",
    "0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52",
    "0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1",
    "0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca",
    "0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676",
    "0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8",
)

_local_url = ""
_run_handle = None
_tx_count = 0


class RpcFailure(RuntimeError):
    def __init__(self, method: str, item: object):
        super().__init__(f"{method}: local JSON-RPC error")
        self.method = method
        self.item = item


def output(*argv: str, cwd: pathlib.Path) -> str:
    return subprocess.check_output(list(argv), cwd=cwd, text=True).strip()


def record(step: str, status: str, **details) -> None:
    if _run_handle is None:
        raise RuntimeError("run log is not open")
    row = {"time": time.strftime("%Y-%m-%dT%H:%M:%S%z"), "step": step, "status": status, **details}
    _run_handle.write(json.dumps(row, sort_keys=True) + "\n")
    _run_handle.flush()
    os.fsync(_run_handle.fileno())
    print(f"{step}: {status}", flush=True)


def rpc(method: str, params: list):
    payload = json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": params}).encode()
    request = urllib.request.Request(_local_url, data=payload, headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(request, timeout=120) as response:
            body = json.loads(response.read())
    except urllib.error.HTTPError as error:
        raw = error.read()
        try:
            body = json.loads(raw)
        except json.JSONDecodeError as decode_error:
            raise RpcFailure(
                method,
                {"http": error.code, "kind": "non-json", "complete_raw_response_hex": "0x" + raw.hex()},
            ) from decode_error
    if "error" in body:
        raise RpcFailure(method, body["error"])
    return body["result"]


def cast(args: argparse.Namespace, *values: str) -> str:
    return output(str(args.cast), *values, cwd=args.lane)


def cast_argument(value: object) -> str:
    if isinstance(value, bool):
        return "true" if value else "false"
    return str(value)


def calldata(args: argparse.Namespace, signature: str, *values) -> str:
    return cast(args, "calldata", signature, *(cast_argument(value) for value in values))


def selector(args: argparse.Namespace, signature: str) -> str:
    return cast(args, "sig", signature).lower()


def keccak_hex(args: argparse.Namespace, value: str) -> str:
    return cast(args, "keccak", value).lower()


def artifact(args: argparse.Namespace, source: str, contract: str) -> dict:
    path = args.scratch / "build/out" / source / f"{contract}.json"
    with path.open(encoding="utf-8") as handle:
        return json.load(handle)


def object_hex(value: str) -> str:
    return value if value.startswith("0x") else "0x" + value


def words(raw: str) -> list[int]:
    payload = raw[2:]
    if len(payload) % 64:
        raise AssertionError("non-word-aligned ABI response")
    return [int(payload[index : index + 64], 16) for index in range(0, len(payload), 64)]


def call_data_raw(to: str, data: str, *, sender: str | None = None, value: int = 0) -> str:
    transaction = {"to": to, "data": data, "value": hex(value)}
    if sender is not None:
        transaction["from"] = sender
    return rpc("eth_call", [transaction, "latest"])


def call_raw(args: argparse.Namespace, to: str, signature: str, *values, sender: str | None = None) -> str:
    return call_data_raw(to, calldata(args, signature, *values), sender=sender)


def signed_word(value: int, bits: int) -> int:
    masked = value & ((1 << bits) - 1)
    sign = 1 << (bits - 1)
    return masked - (1 << bits) if masked & sign else masked


def dynamic_uints(raw: str) -> list[int]:
    decoded = words(raw)
    if len(decoded) < 2 or decoded[0] != 32:
        raise AssertionError("unexpected dynamic uint-array encoding")
    length = decoded[1]
    if len(decoded) != length + 2:
        raise AssertionError("dynamic uint-array length mismatch")
    return decoded[2:]


def canonical_sha256(value: object) -> str:
    encoded = json.dumps(value, sort_keys=True, separators=(",", ":")).encode()
    return hashlib.sha256(encoded).hexdigest()


def write_json_exclusive(path: pathlib.Path, value: object) -> None:
    with path.open("x", encoding="utf-8") as handle:
        json.dump(value, handle, indent=2, sort_keys=True)
        handle.write("\n")
        handle.flush()
        os.fsync(handle.fileno())
    descriptor = os.open(path.parent, os.O_RDONLY | os.O_DIRECTORY)
    try:
        os.fsync(descriptor)
    finally:
        os.close(descriptor)


def uint_call(args: argparse.Namespace, to: str, signature: str, *values) -> int:
    values_out = words(call_raw(args, to, signature, *values))
    if len(values_out) != 1:
        raise AssertionError(f"{signature}: expected one return word")
    return values_out[0]


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


def wait_receipt(tx_hash: str) -> dict:
    deadline = time.monotonic() + 120
    while time.monotonic() < deadline:
        receipt = rpc("eth_getTransactionReceipt", [tx_hash])
        if receipt is not None:
            return receipt
        time.sleep(0.05)
    raise RuntimeError("local receipt timeout")


def send_data(
    sender: str,
    to: str | None,
    data: str,
    label: str,
    *,
    expected_status: int = 1,
) -> dict:
    global _tx_count
    transaction = {"from": sender, "data": data, "gas": hex(MAX_TX_GAS), "value": "0x0"}
    if to is not None:
        transaction["to"] = to
    receipt = wait_receipt(rpc("eth_sendTransaction", [transaction]))
    observed = int(receipt["status"], 16)
    _tx_count += 1
    ok = observed == expected_status
    record(
        label,
        "PASS" if ok else "FAIL",
        observed_status=observed,
        expected_status=expected_status,
        gas_used=int(receipt["gasUsed"], 16),
        transaction_hash=receipt["transactionHash"],
        block_number=int(receipt["blockNumber"], 16),
        complete_receipt=receipt,
    )
    if not ok:
        raise AssertionError(f"{label}: status {observed}, expected {expected_status}")
    transaction_details = rpc("eth_getTransactionByHash", [receipt["transactionHash"]])
    if transaction_details is None:
        raise AssertionError(f"{label}: missing mined transaction details")
    record(f"{label}-transaction", "PASS", complete_transaction=transaction_details)
    return receipt


def send(
    args: argparse.Namespace,
    sender: str,
    to: str,
    signature: str,
    values: tuple,
    label: str,
    *,
    expected_status: int = 1,
) -> dict:
    return send_data(sender, to, calldata(args, signature, *values), label, expected_status=expected_status)


def deploy(args: argparse.Namespace, sender: str, source: str, contract: str, label: str, constructor: str = "") -> str:
    build = artifact(args, source, contract)
    creation = object_hex(build["bytecode"]["object"]) + constructor
    receipt = send_data(sender, None, creation, label)
    address = receipt.get("contractAddress")
    if not address:
        raise AssertionError(f"{label}: missing contract address")
    return address.lower()


def current_timestamp() -> int:
    return int(rpc("eth_getBlockByNumber", ["latest", False])["timestamp"], 16)


def advance_to(timestamp: int, label: str) -> None:
    before = current_timestamp()
    if timestamp > before:
        rpc("evm_setNextBlockTimestamp", [timestamp])
    rpc("evm_mine", [])
    observed = current_timestamp()
    ok = observed >= timestamp
    record(label, "PASS" if ok else "FAIL", before=before, requested=timestamp, observed=observed)
    if not ok:
        raise AssertionError(f"{label}: timestamp boundary not reached")


def advance_seconds(seconds: int, label: str) -> None:
    rpc("evm_increaseTime", [seconds])
    rpc("evm_mine", [])
    rpc("evm_mine", [])
    record(label, "PASS", seconds=seconds, timestamp=current_timestamp())


def next_monday_six_utc(timestamp: int) -> int:
    current = datetime.datetime.fromtimestamp(timestamp, tz=datetime.timezone.utc)
    days = (7 - current.weekday()) % 7
    candidate_date = (current + datetime.timedelta(days=days)).date()
    candidate = datetime.datetime.combine(candidate_date, datetime.time(6, 0), tzinfo=datetime.timezone.utc)
    if candidate.timestamp() <= timestamp:
        candidate += datetime.timedelta(days=7)
    return int(candidate.timestamp())


def find_hex(value: object) -> list[str]:
    found: list[str] = []
    if isinstance(value, str):
        found.extend(item.lower() for item in re.findall(r"0x[0-9a-fA-F]{8,}", value))
    elif isinstance(value, dict):
        for item in value.values():
            found.extend(find_hex(item))
    elif isinstance(value, list):
        for item in value:
            found.extend(find_hex(item))
    return found


def expect_revert_selector(
    args: argparse.Namespace,
    sender: str,
    to: str,
    signature: str,
    values: tuple,
    expected_selector: str,
    label: str,
) -> None:
    try:
        call_raw(args, to, signature, *values, sender=sender)
    except RpcFailure as error:
        matches = [item for item in find_hex(error.item) if item.startswith(expected_selector)]
        ok = bool(matches)
        record(label, "PASS" if ok else "FAIL", expected_selector=expected_selector, selector_matched=ok)
        if not ok:
            raise AssertionError(f"{label}: expected selector absent") from error
        return
    record(label, "FAIL", expected_selector=expected_selector, reason="local-call-succeeded")
    raise AssertionError(f"{label}: local call unexpectedly succeeded")


def manifest_runtime_check(args: argparse.Namespace) -> dict:
    with args.manifest.open(encoding="utf-8") as handle:
        release = json.load(handle)
    contracts = release.get("contracts", {})
    assert_equal("release-runtime-count", len(contracts), 30)
    rows = []
    for name, item in contracts.items():
        address = item["address"].lower()
        actual = keccak_hex(args, rpc("eth_getCode", [address, "latest"]))
        expected = item["runtimeCodeHash"].lower()
        rows.append({"name": name, "address": address, "actual": actual, "expected": expected})
        if actual != expected:
            record("runtime-hashes", "FAIL", rows=rows)
            raise AssertionError(f"runtime mismatch: {name}")
    record("runtime-hashes", "PASS", count=len(rows), rows=rows)
    return release


def install_adapter(args: argparse.Namespace, address: str, source: str, contract: str, label: str) -> None:
    build = artifact(args, source, contract)
    runtime = object_hex(build["deployedBytecode"]["object"])
    rpc("anvil_setCode", [address, runtime])
    observed = rpc("eth_getCode", [address, "latest"])
    assert_equal(label, keccak_hex(args, observed), keccak_hex(args, runtime))


def install_control(args: argparse.Namespace, deployer: str) -> None:
    original = rpc("eth_getCode", [BOOK, "latest"])
    constructor = ROUTER[2:].rjust(64, "0") + ENGINE[2:].rjust(64, "0")
    deployed = deploy(
        args,
        deployer,
        "PositionProtectionBook.sol",
        "PositionProtectionBook",
        "deploy-control-position-protection-book",
        constructor,
    )
    runtime = rpc("eth_getCode", [deployed, "latest"])
    assert_gt("control-runtime-nonempty", len(runtime), 2)
    assert_equal("control-runtime-differs", keccak_hex(args, runtime) == keccak_hex(args, original), False)
    assert_equal("control-router-immutable", address_call(args, deployed, "ROUTER()"), ROUTER)
    assert_equal("control-engine-immutable", address_call(args, deployed, "ENGINE()"), ENGINE)
    rpc("anvil_setCode", [BOOK, runtime])
    control_runtime_hash = keccak_hex(args, runtime)
    assert_equal("control-runtime-installed", keccak_hex(args, rpc("eth_getCode", [BOOK, "latest"])), control_runtime_hash)
    assert_equal("router-book-binding", address_call(args, ROUTER, "positionProtectionBook()"), BOOK)
    assert_equal("installed-control-router", address_call(args, BOOK, "ROUTER()"), ROUTER)
    assert_equal("installed-control-engine", address_call(args, BOOK, "ENGINE()"), ENGINE)

    # Re-verify the full release runtime set after the one preregistered Book exception.
    with args.manifest.open(encoding="utf-8") as handle:
        release = json.load(handle)
    rows = []
    book_exceptions = 0
    for name, item in release.get("contracts", {}).items():
        address = item["address"].lower()
        actual = keccak_hex(args, rpc("eth_getCode", [address, "latest"]))
        expected = item["runtimeCodeHash"].lower()
        if address == BOOK:
            book_exceptions += 1
            status = "PREREGISTERED_CONTROL_EXCEPTION"
            if actual == expected or actual != control_runtime_hash:
                record("control-post-install-runtime-hashes", "FAIL", rows=rows)
                raise AssertionError("control Book runtime exception mismatch")
        else:
            status = "EXACT_RELEASE_RUNTIME"
            if actual != expected:
                record("control-post-install-runtime-hashes", "FAIL", rows=rows)
                raise AssertionError(f"post-control runtime mismatch: {name}")
        rows.append({"name": name, "address": address, "actual": actual, "expected": expected, "status": status})
    if len(rows) != 30 or book_exceptions != 1:
        record(
            "control-post-install-runtime-hashes",
            "FAIL",
            runtime_count=len(rows),
            book_exceptions=book_exceptions,
            rows=rows,
        )
        raise AssertionError("post-control runtime exception census mismatch")
    record(
        "control-post-install-runtime-hashes",
        "PASS",
        runtime_count=len(rows),
        book_exceptions=book_exceptions,
        rows=rows,
    )


def view_record(args: argparse.Namespace, to: str, signature: str, *values) -> dict[str, object]:
    raw = call_raw(args, to, signature, *values)
    return {"signature": signature, "arguments": [cast_argument(value) for value in values], "raw": raw, "words": words(raw)}


def capture_creation_boundary(
    args: argparse.Namespace,
    controllers: list[str],
    victim: str,
    keeper: str,
    outsider: str,
    create_data: str,
) -> dict[str, object]:
    block = rpc("eth_getBlockByNumber", ["latest", False])
    oracle = address_call(args, ROUTER, "pletherOracle()")
    admin = address_call(args, ROUTER, "admin()")
    planner = address_call(args, ENGINE, "planner()")
    pyth = address_call(args, oracle, "pyth()")
    assert_equal("boundary-pyth-binding", pyth, PYTH)

    position_view = view_record(args, ENGINE, "positions(address)", victim)
    position_words = position_view["words"]
    if not isinstance(position_words, list) or len(position_words) != 7:
        raise AssertionError("boundary position tuple length")
    carry_state = view_record(args, ENGINE, "positionCarryState(address)", victim)
    risk_params = view_record(args, ENGINE, "riskParams()")
    risk_words = risk_params["words"]
    if not isinstance(risk_words, list) or len(risk_words) != 10:
        raise AssertionError("boundary risk tuple length")

    fad_window = bool(uint_call(args, ENGINE, "isFadWindow()"))
    oracle_frozen = bool(uint_call(args, ENGINE, "isOracleFrozen()"))
    unsettled_carry = uint_call(args, ENGINE, "unsettledCarryAssets(address)", victim)
    trader_claim = uint_call(args, ENGINE, "traderClaimBalanceAssets(address)", victim)
    entry_cost = uint_call(args, ENGINE, "positionEntryCostAssetAtoms(address)", victim)
    cap_price = uint_call(args, ENGINE, "CAP_PRICE()")
    active_margin_bps = risk_words[4] if fad_window else risk_words[2]
    required_margin_bps = max(risk_words[3], active_margin_bps)
    vpi_accrued = signed_word(position_words[6], 256)
    negative_vpi_target = -vpi_accrued if vpi_accrued < 0 else 0

    isolation = view_record(args, CLEARINGHOUSE, "getPnlIsolationBuckets(address)", victim)
    isolation_words = isolation["words"]
    if not isinstance(isolation_words, list) or len(isolation_words) != 8:
        raise AssertionError("boundary isolation tuple length")

    accepted_basket = view_record(args, oracle, "getAcceptedBasketMark()")
    accepted_times_raw = call_raw(args, oracle, "getAcceptedFeedPublishTimes()")
    accepted_times = dynamic_uints(accepted_times_raw)
    if len(accepted_times) != len(FEED_IDS):
        raise AssertionError("accepted feed count mismatch")
    latest_mark_snapshot = view_record(args, oracle, "getLatestPrice(uint8)", 1)
    close_policy = view_record(args, oracle, "getOrderExecutionPolicy(bool)", True)

    feeds = []
    for index, expected_feed_id in enumerate(FEED_IDS):
        configured_raw = call_raw(args, oracle, "pythFeedIds(uint256)", index).lower()
        assert_equal(f"boundary-feed-{index}-id", configured_raw, expected_feed_id)
        observation = view_record(args, pyth, "getPriceUnsafe(bytes32)", configured_raw)
        observation_words = observation["words"]
        if not isinstance(observation_words, list) or len(observation_words) != 4:
            raise AssertionError(f"feed {index} observation tuple length")
        feeds.append(
            {
                "index": index,
                "feed_id": configured_raw,
                "quantity": uint_call(args, oracle, "quantities(uint256)", index),
                "base_price": uint_call(args, oracle, "basePrices(uint256)", index),
                "inverted": bool(uint_call(args, oracle, "inversions(uint256)", index)),
                "accepted_publish_time": accepted_times[index],
                "pyth_observation": observation,
                "decoded_observation": {
                    "price": signed_word(observation_words[0], 64),
                    "confidence": observation_words[1],
                    "exponent": signed_word(observation_words[2], 32),
                    "publish_time": observation_words[3],
                },
            }
        )

    today = int(block["timestamp"], 16) // 86_400
    runtime_addresses = {
        "settlement_token": TOKEN,
        "clearinghouse": CLEARINGHOUSE,
        "engine": ENGINE,
        "router": ROUTER,
        "position_protection_book": BOOK,
        "plether_oracle": oracle,
        "pyth_local_adapter": pyth,
        "arbsys_local_adapter": ARB_SYS,
        "planner": planner,
        "router_admin": admin,
    }
    runtime_hashes = {
        name: {"address": address, "runtime_keccak": keccak_hex(args, rpc("eth_getCode", [address, "latest"]))}
        for name, address in runtime_addresses.items()
    }

    return {
        "chain": {
            "chain_id": int(rpc("eth_chainId", []), 16),
            "block_number": int(block["number"], 16),
            "block_hash": block["hash"],
            "timestamp": int(block["timestamp"], 16),
        },
        "transaction": {
            "from": controllers[0],
            "to": victim,
            "data": create_data,
            "value": 0,
        },
        "controllers": {"victim": controllers[0], "keeper": controllers[1], "outsider": controllers[2]},
        "actors": {"victim": victim, "keeper": keeper, "outsider": outsider},
        "position": {
            "tuple": position_view,
            "decoded": {
                "size": position_words[0],
                "margin": position_words[1],
                "entry_price": position_words[2],
                "max_profit_assets": position_words[3],
                "side": position_words[4],
                "last_update_time": position_words[5],
                "vpi_accrued": vpi_accrued,
            },
            "carry_state": carry_state,
            "entry_cost_asset_atoms": entry_cost,
            "unsettled_carry_assets": unsettled_carry,
            "trader_claim_balance_assets": trader_claim,
        },
        "clearinghouse": {
            "pnl_isolation_buckets": isolation,
            "decoded": {
                "settlement_balance_assets": isolation_words[0],
                "pnl_pledge_assets": isolation_words[1],
                "liquidation_reserve_assets": isolation_words[2],
                "order_margin_assets": isolation_words[3],
                "action_reserve_assets": isolation_words[4],
                "vpi_rebate_reserve_assets": isolation_words[5],
                "total_locked_assets": isolation_words[6],
                "free_settlement_assets": isolation_words[7],
            },
            "active_order_reservations": view_record(
                args, CLEARINGHOUSE, "getAccountReservationSummary(address)", victim
            ),
            "total_bounty_reservations_assets": uint_call(
                args, CLEARINGHOUSE, "totalBountyReservationsAssets(address)", victim
            ),
            "token_balance_atoms": uint_call(args, TOKEN, "balanceOf(address)", victim),
        },
        "router_and_protection": {
            "pending_order_count": uint_call(args, ROUTER, "pendingOrderCounts(address)", victim),
            "pending_head_order_id": uint_call(args, ROUTER, "accountHeadOrderId(address)", victim),
            "active_protection_id": uint_call(args, BOOK, "activePositionProtectionId(address)", victim),
            "next_commit_id": uint_call(args, ROUTER, "nextCommitId()"),
            "configured_protection_bounty_assets": uint_call(
                args, ROUTER, "positionProtectionTriggerBountyAssets()"
            ),
        },
        "exact_price_risk_inputs": {
            "planner": planner,
            "risk_params": risk_params,
            "fad_window": fad_window,
            "active_margin_bps": active_margin_bps,
            "required_margin_bps": required_margin_bps,
            "cached_mark_price": uint_call(args, ENGINE, "lastMarkPrice()"),
            "cached_mark_time": uint_call(args, ENGINE, "lastMarkTime()"),
            "cap_price": cap_price,
            "price_collateral_assets": position_words[1] + trader_claim,
            "negative_vpi_reserve_target_assets": negative_vpi_target,
            "vpi_rebate_reserve_assets": isolation_words[5],
            "unsettled_carry_assets": unsettled_carry,
            "position_entry_cost_asset_atoms": entry_cost,
        },
        "oracle": {
            "address": oracle,
            "accepted_basket": accepted_basket,
            "accepted_feed_publish_times_raw": accepted_times_raw,
            "accepted_feed_publish_times": accepted_times,
            "latest_mark_refresh_snapshot": latest_mark_snapshot,
            "close_policy": close_policy,
            "feeds": feeds,
            "configuration": {
                "order_execution_staleness_limit": uint_call(args, oracle, "orderExecutionStalenessLimit()"),
                "liquidation_staleness_limit": uint_call(args, oracle, "liquidationStalenessLimit()"),
                "basket_max_confidence_ratio_bps": uint_call(args, oracle, "basketMaxConfidenceRatioBps()"),
                "order_settlement_window": uint_call(args, oracle, "orderSettlementWindow()"),
                "max_component_publish_time_divergence": uint_call(
                    args, oracle, "maxComponentPublishTimeDivergence()"
                ),
                "adverse_confidence_multiplier_bps": uint_call(
                    args, oracle, "adverseConfidenceMultiplierBps()"
                ),
            },
        },
        "calendar_and_safety": {
            "is_fad_window": fad_window,
            "is_oracle_frozen": oracle_frozen,
            "today_utc_day": today,
            "today_override": bool(uint_call(args, ENGINE, "fadDayOverrides(uint256)", today)),
            "tomorrow_override": bool(uint_call(args, ENGINE, "fadDayOverrides(uint256)", today + 1)),
            "fad_runway_seconds": uint_call(args, ENGINE, "fadRunwaySeconds()"),
            "fad_max_staleness": uint_call(args, ENGINE, "fadMaxStaleness()"),
            "engine_mark_staleness_limit": uint_call(args, ENGINE, "engineMarkStalenessLimit()"),
            "degraded_mode": bool(uint_call(args, ENGINE, "degradedMode()")),
            "router_admin": admin,
            "router_paused": bool(uint_call(args, admin, "paused()")),
            "risk_off_order_cutoff": uint_call(args, admin, "riskOffOrderCutoff()"),
        },
        "runtime_hashes": runtime_hashes,
    }


def candidate_state(args: argparse.Namespace, victim: str, protection_id: int, child: int) -> dict[str, object]:
    lifecycle = address_call(args, ROUTER, "lifecycleBook()")
    return {
        "protection": protection(args, protection_id),
        "close_limits": close_limits(args, protection_id),
        "position": position(args, victim),
        "settlement_balance": uint_call(args, CLEARINGHOUSE, "balanceAssets(address)", victim),
        "locked_margin": uint_call(args, CLEARINGHOUSE, "lockedMarginAssets(address)", victim),
        "action_reserve": uint_call(args, CLEARINGHOUSE, "actionReserveAssets(address)", victim),
        "vpi_rebate_reserve": uint_call(args, CLEARINGHOUSE, "vpiRebateReserveAssets(address)", victim),
        "pending_count": uint_call(args, ROUTER, "pendingOrderCounts(address)", victim),
        "pending_head": uint_call(args, ROUTER, "accountHeadOrderId(address)", victim),
        "child_lifecycle": uint_call(args, lifecycle, "lifecycleStatus(uint64)", child),
    }


def assert_state_unchanged(label: str, before: dict[str, object], after: dict[str, object]) -> None:
    before_hash = canonical_sha256(before)
    after_hash = canonical_sha256(after)
    ok = before == after
    record(label, "PASS" if ok else "FAIL", before_sha256=before_hash, after_sha256=after_hash)
    if not ok:
        raise AssertionError(f"{label}: state changed")


def actor_send(args: argparse.Namespace, sender: str, actor: str, signature: str, values: tuple, label: str, *, expected_status: int = 1) -> dict:
    return send(args, sender, actor, signature, values, label, expected_status=expected_status)


def protection(args: argparse.Namespace, protection_id: int) -> list[int]:
    result = words(call_raw(args, BOOK, "getPositionProtection(uint64)", protection_id))
    if len(result) != 15:
        raise AssertionError("unexpected protection view length")
    return result


def close_limits(args: argparse.Namespace, protection_id: int) -> list[int]:
    result = words(call_raw(args, BOOK, "closeFinancialLimits(uint64)", protection_id))
    if len(result) != 4:
        raise AssertionError("unexpected close limits length")
    return result


def position(args: argparse.Namespace, account: str) -> list[int]:
    result = words(call_raw(args, ENGINE, "positions(address)", account))
    if len(result) != 7:
        raise AssertionError("unexpected position length")
    return result


def order_deadline(args: argparse.Namespace, order_id: int) -> int:
    result = words(call_raw(args, address_call(args, ROUTER, "lifecycleBook()"), "orderTiming(uint64)", order_id))
    if len(result) != 4:
        raise AssertionError("unexpected order timing length")
    return result[3]


def pending_notional_limit(args: argparse.Namespace, order_id: int) -> int:
    result = words(call_raw(args, address_call(args, ROUTER, "lifecycleBook()"), "pendingPolicy(uint64)", order_id))
    if len(result) != 13:
        raise AssertionError("unexpected pending policy length")
    return result[5]


def assert_triggered(args: argparse.Namespace, victim: str, protection_id: int, child: int, label: str) -> None:
    view = protection(args, protection_id)
    assert_equal(f"{label}-status", view[14], STATUS_TRIGGERED)
    assert_equal(f"{label}-linked", view[2], child)
    assert_equal(f"{label}-pending-count", uint_call(args, ROUTER, "pendingOrderCounts(address)", victim), 1)
    assert_equal(f"{label}-queue-head", uint_call(args, ROUTER, "accountHeadOrderId(address)", victim), child)


def run_target(args: argparse.Namespace, accounts: list[str]) -> dict[str, object]:
    if len(accounts) < 3:
        raise RuntimeError("three distinct local development accounts are required")
    controllers = [account.lower() for account in accounts[:3]]
    assert_equal("distinct-role-controllers", len(set(controllers)), 3)

    install_adapter(args, PYTH, "LocalPythAdapter.sol", "LocalPythAdapter", "local-pyth-adapter")
    install_adapter(args, ARB_SYS, "LocalArbSysAdapter.sol", "LocalArbSysAdapter", "local-arbsys-adapter")
    rpc("evm_mine", [])
    assert_equal("local-arbsys-block", uint_call(args, ARB_SYS, "arbBlockNumber()"), int(rpc("eth_blockNumber", []), 16))
    if args.mode == "control":
        install_control(args, controllers[0])

    advance_to(next_monday_six_utc(current_timestamp()), "monday-six-utc")
    assert_equal(
        "local-block-timestamp-interval",
        rpc("anvil_setBlockTimestampInterval", [1]),
        None,
    )
    victim = deploy(args, controllers[0], "AF011Actors.sol", "AF011VictimActor", "deploy-victim")
    keeper = deploy(args, controllers[1], "AF011Actors.sol", "AF011KeeperActor", "deploy-keeper")
    outsider = deploy(args, controllers[2], "AF011Actors.sol", "AF011OutsiderHelper", "deploy-outsider")
    assert_equal("distinct-actor-contracts", len({victim, keeper, outsider}), 3)
    for label, actor, controller in (
        ("victim", victim, controllers[0]),
        ("keeper", keeper, controllers[1]),
        ("outsider", outsider, controllers[2]),
    ):
        assert_equal(f"{label}-controller", address_call(args, actor, "controller()"), controller)

    actor_send(args, controllers[1], keeper, "updateMarkPrice(uint256)", (MARK_PRICE,), "initial-refresh-mark")
    assert_equal("mark-price", uint_call(args, ENGINE, "lastMarkPrice()"), MARK_PRICE)
    actor_send(args, controllers[0], victim, "fundTrader(uint256)", (TRADER_FUNDS,), "victim-fund")
    actor_send(
        args,
        controllers[0],
        victim,
        "commitOpen(uint256,uint256)",
        (POSITION_SIZE, POSITION_MARGIN),
        "victim-open-commit",
    )
    open_id = uint_call(args, victim, "lastOrderId()")
    advance_seconds(7, "open-execution-wait")
    actor_send(args, controllers[1], keeper, "executeOrder(uint64,uint256)", (open_id, MARK_PRICE), "victim-open-execute")
    assert_equal("victim-position-size", position(args, victim)[0], POSITION_SIZE)

    actor_send(
        args,
        controllers[1],
        keeper,
        "updateMarkPrice(uint256)",
        (MARK_PRICE,),
        "post-open-refresh-mark",
    )
    assert_equal("post-open-mark-price", uint_call(args, ENGINE, "lastMarkPrice()"), MARK_PRICE)

    create_signature = "createProtection(uint256,uint256,uint256,uint256,uint256,uint256)"
    create_values = (TRIGGER_PRICE, 0, INITIAL_NOTIONAL_LIMIT, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT)
    create_data = calldata(args, create_signature, *create_values)
    boundary_before = capture_creation_boundary(args, controllers, victim, keeper, outsider, create_data)
    before_path = args.evidence_dir / "creation-boundary-before.json"
    after_path = args.evidence_dir / "creation-boundary-after-diagnostic.json"
    diagnostic_path = args.evidence_dir / "creation-diagnostic.json"
    raw_success_path = args.evidence_dir / "creation-diagnostic-raw-success.json"
    raw_error_path = args.evidence_dir / "creation-diagnostic-raw-error.json"
    write_json_exclusive(before_path, boundary_before)
    before_hash = canonical_sha256(boundary_before)
    diagnostic: dict[str, object] = {
        "transaction": boundary_before["transaction"],
        "boundary_before_sha256": before_hash,
        "status": "FAIL_INCOMPLETE",
    }
    try:
        diagnostic_raw = call_data_raw(victim, create_data, sender=controllers[0], value=0)
    except RpcFailure as error:
        # Preserve the complete primary failure before any secondary capture or decoding can fail.
        write_json_exclusive(
            raw_error_path,
            {
                "status": "RAW_RPC_ERROR_PRESERVED_BEFORE_SECONDARY_CAPTURE",
                "transaction": boundary_before["transaction"],
                "boundary_before_sha256": before_hash,
                "method": error.method,
                "complete_rpc_error": error.item,
            },
        )
        boundary_after = capture_creation_boundary(args, controllers, victim, keeper, outsider, create_data)
        write_json_exclusive(after_path, boundary_after)
        hex_candidates = find_hex(error.item)
        selector_names = {
            selector(args, "OrderRouter__InsufficientFreeEquity()"): "OrderRouter__InsufficientFreeEquity",
            selector(args, "MarginClearinghouse__InsufficientFreeEquity()"): "MarginClearinghouse__InsufficientFreeEquity",
            selector(args, "OrderRouter__ProtectionTriggerAlreadyMet()"): "OrderRouter__ProtectionTriggerAlreadyMet",
            selector(args, "OrderRouter__ProtectionCommitDisabled()"): "OrderRouter__ProtectionCommitDisabled",
        }
        observed_selectors = sorted({candidate[:10] for candidate in hex_candidates if len(candidate) >= 10})
        diagnostic.update(
            {
                "status": "ERROR_VULNERABLE_PRECONDITION_FAILURE_CONSUMED",
                "complete_rpc_error": error.item,
                "raw_hex_candidates": hex_candidates,
                "observed_selectors": observed_selectors,
                "unambiguous_known_selector_names": [
                    selector_names[item] for item in observed_selectors if item in selector_names
                ],
                "boundary_after_sha256": canonical_sha256(boundary_after),
            }
        )
        write_json_exclusive(diagnostic_path, diagnostic)
        record(
            "creation-diagnostic-call",
            "FAIL",
            complete_rpc_error=error.item,
            observed_selectors=observed_selectors,
        )
        raise RuntimeError("creation diagnostic failed; vulnerable allowance consumed; no candidate or control") from error
    # Preserve the undecoded return before ABI-shape validation or secondary capture.
    write_json_exclusive(
        raw_success_path,
        {
            "status": "RAW_SUCCESS_PRESERVED_BEFORE_DECODE_AND_SECONDARY_CAPTURE",
            "transaction": boundary_before["transaction"],
            "boundary_before_sha256": before_hash,
            "return_data": diagnostic_raw,
        },
    )
    diagnostic_words = words(diagnostic_raw)
    if len(diagnostic_words) != 1:
        raise AssertionError("creation diagnostic returned an unexpected ABI shape")
    boundary_after = capture_creation_boundary(args, controllers, victim, keeper, outsider, create_data)
    write_json_exclusive(after_path, boundary_after)
    after_hash = canonical_sha256(boundary_after)
    unchanged = boundary_before == boundary_after
    record(
        "creation-diagnostic-call",
        "PASS",
        return_data=diagnostic_raw,
        returned_protection_id=diagnostic_words[0],
    )
    record(
        "creation-diagnostic-state-unchanged",
        "PASS" if unchanged else "FAIL",
        before_sha256=before_hash,
        after_sha256=after_hash,
    )
    if not unchanged:
        raise AssertionError("creation diagnostic changed captured state")
    diagnostic.update(
        {
            "status": "PASS",
            "return_data": diagnostic_raw,
            "returned_protection_id": diagnostic_words[0],
            "boundary_after_sha256": after_hash,
            "state_unchanged": True,
        }
    )
    write_json_exclusive(diagnostic_path, diagnostic)

    # No mutating transaction occurs between the same-calldata diagnostic and this mined call.
    send_data(controllers[0], victim, create_data, "victim-create-protection")
    protection_id = uint_call(args, victim, "lastProtectionId()")
    assert_equal("creation-diagnostic-id-match", protection_id, diagnostic_words[0])
    assert_equal("active-protection-id", uint_call(args, BOOK, "activePositionProtectionId(address)", victim), protection_id)
    assert_equal("protection-armed", protection(args, protection_id)[14], STATUS_ARMED)
    assert_equal("protection-no-linked-child-before-trigger", protection(args, protection_id)[2], 0)
    assert_equal("initial-close-limits", close_limits(args, protection_id), [INITIAL_NOTIONAL_LIMIT, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT])
    bounty = words(call_raw(args, CLEARINGHOUSE, "getBountyReservation(uint8,uint64)", PROTECTION_BOUNTY_KIND, protection_id))
    if len(bounty) != 4:
        raise AssertionError("protection bounty tuple length")
    assert_equal("protection-bounty-account", "0x" + hex(bounty[0])[2:].rjust(64, "0")[-40:], victim)
    assert_equal(
        "protection-bounty-amount",
        bounty[1],
        uint_call(args, ROUTER, "positionProtectionTriggerBountyAssets()"),
    )
    assert_equal("protection-bounty-active-state", bounty[2], 1)
    assert_equal(
        "protection-bounty-total",
        uint_call(args, CLEARINGHOUSE, "totalBountyReservationsAssets(address)", victim),
        bounty[1],
    )
    advance_seconds(2, "trigger-freshness-wait")
    actor_send(
        args,
        controllers[1],
        keeper,
        "triggerProtection(uint64,uint256)",
        (protection_id, TRIGGER_PRICE),
        "keeper-trigger-protection",
    )
    first_child = protection(args, protection_id)[2]
    assert_triggered(args, victim, protection_id, first_child, "first-child")

    advance_to(order_deadline(args, first_child) + 1, "first-child-deadline")
    actor_send(args, controllers[1], keeper, "expireOrder(uint64)", (first_child,), "keeper-expire-first-child")
    assert_equal("first-expiry-latched", protection(args, protection_id)[14], STATUS_LATCHED)
    actor_send(
        args,
        controllers[0],
        victim,
        "amendLimits(uint64,uint256,uint256,uint256,uint256)",
        (protection_id, AMENDED_NOTIONAL_LIMIT, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT),
        "owner-amend-latched-limits",
    )
    assert_equal("amended-close-limits", close_limits(args, protection_id), [AMENDED_NOTIONAL_LIMIT, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT])

    if args.mode == "vulnerable":
        actor_send(args, controllers[2], outsider, "retryOnly(uint64)", (protection_id,), "outsider-initial-retry")
        child = uint_call(args, outsider, "lastOrderId()")
    else:
        actor_send(args, controllers[0], victim, "retryProtection(uint64)", (protection_id,), "owner-initial-retry")
        child = uint_call(args, victim, "lastOrderId()")
    assert_gt("retry-child-newer", child, first_child)
    assert_triggered(args, victim, protection_id, child, "retry-child")
    assert_equal("retry-child-amended-limit", pending_notional_limit(args, child), AMENDED_NOTIONAL_LIMIT)

    position_before = position(args, victim)
    balance_before = uint_call(args, CLEARINGHOUSE, "balanceAssets(address)", victim)
    reserve_before = uint_call(args, CLEARINGHOUSE, "actionReserveAssets(address)", victim)
    assert_equal("outsider-helper-zero-token-before", uint_call(args, TOKEN, "balanceOf(address)", outsider), 0)
    advance_to(order_deadline(args, child) + 1, "target-child-deadline")

    unauthorized = selector(args, "OrderRouter__Unauthorized()")
    not_latched = selector(args, "OrderRouter__ProtectionNotLatched()")
    if args.mode == "vulnerable":
        actor_send(
            args,
            controllers[2],
            outsider,
            "expireAndRetry(uint64,uint64)",
            (child, protection_id),
            "outsider-atomic-expire-retry",
        )
        replacement = uint_call(args, outsider, "lastOrderId()")
        assert_gt("atomic-replacement-newer", replacement, child)
        assert_triggered(args, victim, protection_id, replacement, "atomic-replacement")
        assert_equal("atomic-replacement-amended-limit", pending_notional_limit(args, replacement), AMENDED_NOTIONAL_LIMIT)
        amendment_revert_before = candidate_state(args, victim, protection_id, replacement)
        expect_revert_selector(
            args,
            controllers[0],
            victim,
            "amendLimits(uint64,uint256,uint256,uint256,uint256)",
            (protection_id, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT),
            not_latched,
            "owner-amendment-local-revert-selector",
        )
        failed = actor_send(
            args,
            controllers[0],
            victim,
            "amendLimits(uint64,uint256,uint256,uint256,uint256)",
            (protection_id, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT, OTHER_LIMIT),
            "owner-amendment-failed-transaction",
            expected_status=0,
        )
        assert_gt("owner-amendment-failed-gas", int(failed["gasUsed"], 16), 0)
        amendment_revert_after = candidate_state(args, victim, protection_id, replacement)
        assert_state_unchanged(
            "owner-amendment-revert-state-unchanged", amendment_revert_before, amendment_revert_after
        )
        terminal_child = replacement
    else:
        control_rollback_before = candidate_state(args, victim, protection_id, child)
        expect_revert_selector(
            args,
            controllers[2],
            outsider,
            "expireAndRetry(uint64,uint64)",
            (child, protection_id),
            unauthorized,
            "control-outsider-local-revert-selector",
        )
        failed = actor_send(
            args,
            controllers[2],
            outsider,
            "expireAndRetry(uint64,uint64)",
            (child, protection_id),
            "control-outsider-failed-transaction",
            expected_status=0,
        )
        assert_gt("control-outsider-failed-gas", int(failed["gasUsed"], 16), 0)
        control_rollback_after = candidate_state(args, victim, protection_id, child)
        assert_state_unchanged(
            "control-outsider-atomic-rollback-state-unchanged", control_rollback_before, control_rollback_after
        )
        assert_triggered(args, victim, protection_id, child, "control-atomic-rollback")
        lifecycle = address_call(args, ROUTER, "lifecycleBook()")
        assert_equal("control-child-still-pending", uint_call(args, lifecycle, "lifecycleStatus(uint64)", child), LIFECYCLE_PENDING)
        actor_send(args, controllers[0], victim, "expireOrder(uint64)", (child,), "owner-expire-control-child")
        assert_equal("control-owner-expiry-latched", protection(args, protection_id)[14], STATUS_LATCHED)
        actor_send(args, controllers[0], victim, "retryProtection(uint64)", (protection_id,), "owner-retry-after-control")
        terminal_child = uint_call(args, victim, "lastOrderId()")
        assert_gt("control-owner-replacement-newer", terminal_child, child)
        assert_triggered(args, victim, protection_id, terminal_child, "control-owner-replacement")

    assert_equal("position-unchanged", position(args, victim), position_before)
    assert_equal("settlement-balance-unchanged", uint_call(args, CLEARINGHOUSE, "balanceAssets(address)", victim), balance_before)
    assert_equal("action-reserve-unchanged", uint_call(args, CLEARINGHOUSE, "actionReserveAssets(address)", victim), reserve_before)
    assert_equal("outsider-helper-zero-token-after", uint_call(args, TOKEN, "balanceOf(address)", outsider), 0)
    return {
        "mode": args.mode,
        "status": "PASS_ATTEMPT_CONSUMED_AND_PRESERVED",
        "controllers": {"victim": controllers[0], "keeper": controllers[1], "outsider": controllers[2]},
        "actors": {"victim": victim, "keeper": keeper, "outsider": outsider},
        "protection_id": protection_id,
        "first_child": first_child,
        "target_child": child,
        "terminal_child": terminal_child,
        "transaction_count": _tx_count,
    }


def parse_args() -> argparse.Namespace:
    lane = pathlib.Path(__file__).resolve().parent.parent
    root = lane.parents[2]
    parser = argparse.ArgumentParser()
    parser.add_argument("--mode", choices=sorted(LOCAL_URLS), required=True)
    parser.add_argument("--evidence-dir", type=pathlib.Path, required=True)
    parser.add_argument("--ack-attempt-token", required=True)
    parser.add_argument("--repo", type=pathlib.Path, default=root / "plether-engagement-2026-11/source/audit-comp-plether")
    parser.add_argument(
        "--scratch",
        type=pathlib.Path,
        default=root / ".scratch/plether-af011-final-successor-2026-11-06-codex-r06",
    )
    parser.add_argument("--cast", type=pathlib.Path, required=True)
    args = parser.parse_args()
    if args.ack_attempt_token != f"AF011_R06_{args.mode.upper()}_LOCAL_ATTEMPT":
        parser.error("exact launcher-to-target attempt token is required")
    args.lane = lane
    args.repo = args.repo.resolve()
    args.scratch = args.scratch.resolve()
    args.evidence_dir = args.evidence_dir.resolve()
    args.manifest = args.repo / "deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json"
    return args


def main() -> int:
    global _local_url, _run_handle
    args = parse_args()
    _local_url = LOCAL_URLS[args.mode]
    result_path = args.evidence_dir / "target-result.json"
    status = {"mode": args.mode, "status": "FAIL_ATTEMPT_CONSUMED_AND_PRESERVED"}
    with (args.evidence_dir / "target-events.jsonl").open("x", encoding="utf-8") as handle:
        _run_handle = handle
        try:
            assert_equal("chain-id", int(rpc("eth_chainId", []), 16), CHAIN_ID)
            assert_equal("fork-block", int(rpc("eth_blockNumber", []), 16), FORK_BLOCK)
            manifest_runtime_check(args)
            accounts = rpc("eth_accounts", [])
            status = run_target(args, accounts)
        except Exception as error:
            status["error_type"] = type(error).__name__
            record("target-terminal", "FAIL_ATTEMPT_CONSUMED_AND_PRESERVED", error_type=type(error).__name__)
            write_json_exclusive(result_path, status)
            raise
        else:
            record("target-terminal", "PASS_ATTEMPT_CONSUMED_AND_PRESERVED", transaction_count=_tx_count)
            write_json_exclusive(result_path, status)
    print(f"AF011 {args.mode.upper()} PASS / ATTEMPT CONSUMED AND PRESERVED")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

<!--AF011Actors.sol-->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IAF011MockSettlementToken {
    function mint(address account, uint256 amount) external;
    function approve(address spender, uint256 amount) external returns (bool);
}

interface IAF011MarginClearinghouse {
    function deposit(address account, uint256 amount) external;
}

interface IAF011ReleaseRouter {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData) external payable;
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory result);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function maxExecutionWindowSeconds() external view returns (uint256);
    function lifecycleBook() external view returns (IOrderLifecycleBook);
}

/// @notice Ordinary unprivileged account for AF-011 on a local fork of the pinned deployment.
/// @dev No admin call, impersonation hook, storage write, balance write, or arbitrary-call primitive exists here.
contract AF011VictimActor {
    address internal constant TOKEN = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant BOOK = 0x81E3b03Cf29865b6c0f85925839567e2Bc61073D;

    address private immutable CONTROLLER;
    uint256 public nonce;
    uint64 public lastOrderId;
    uint64 public lastProtectionId;

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

    function fundTrader(uint256 amount) external onlyController {
        IAF011MockSettlementToken(TOKEN).mint(address(this), amount);
        require(IAF011MockSettlementToken(TOKEN).approve(CLEARINGHOUSE, amount), "clearinghouse approval");
        IAF011MarginClearinghouse(CLEARINGHOUSE).deposit(address(this), amount);
    }

    function commitOpen(uint256 size, uint256 margin) external onlyController returns (uint64 orderId) {
        uint256 window = IAF011ReleaseRouter(ROUTER).maxExecutionWindowSeconds();
        require(window <= type(uint32).max, "window overflow");
        require(block.timestamp <= type(uint64).max - window, "submit time overflow");
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode(address(this), ++nonce, size, margin));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = false;
        request.targetPrice = 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            // forge-lint: disable-next-line(unsafe-typecast)
            submitBy: uint64(block.timestamp + window),
            // forge-lint: disable-next-line(unsafe-typecast)
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 1,
            expectedConfigHash: IAF011ReleaseRouter(ROUTER).lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        orderId = IAF011ReleaseRouter(ROUTER).commitOrder(request);
        lastOrderId = orderId;
    }

    function createProtection(
        uint256 takeProfit,
        uint256 stopLoss,
        uint256 maxExecutionNotionalAssets,
        uint256 maxGrossAccountDebitAssets,
        uint256 maxActionChargeAssets,
        uint256 maxExplicitFeesAssets
    ) external onlyController returns (uint64 protectionId) {
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: maxExecutionNotionalAssets,
            maxGrossAccountDebitAssets: maxGrossAccountDebitAssets,
            maxActionChargeAssets: maxActionChargeAssets,
            maxExplicitFeesAssets: maxExplicitFeesAssets
        });
        protectionId = IPositionProtectionActions(BOOK).createPositionProtection(params);
        lastProtectionId = protectionId;
    }

    function amendLimits(
        uint64 protectionId,
        uint256 maxExecutionNotionalAssets,
        uint256 maxGrossAccountDebitAssets,
        uint256 maxActionChargeAssets,
        uint256 maxExplicitFeesAssets
    ) external onlyController {
        IPositionProtectionActions(BOOK)
            .amendLatchedPositionProtectionCloseLimits(
                protectionId,
                PositionProtectionTypes.CloseFinancialLimits({
                    maxExecutionNotionalAssets: maxExecutionNotionalAssets,
                    maxGrossAccountDebitAssets: maxGrossAccountDebitAssets,
                    maxActionChargeAssets: maxActionChargeAssets,
                    maxExplicitFeesAssets: maxExplicitFeesAssets
                })
            );
    }

    function retryProtection(uint64 protectionId) external onlyController returns (uint64 orderId) {
        orderId = IPositionProtectionActions(BOOK).retryPositionProtectionClose(protectionId);
        lastOrderId = orderId;
    }

    function expireOrder(uint64 orderId) external onlyController {
        OrderV3Types.ExecutionResult memory result = IAF011ReleaseRouter(ROUTER).expireOrder(orderId);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "expiry did not terminalize");
        require(result.terminalReason == OrderV3Types.TerminalReason.Expired, "wrong terminal reason");
    }
}

/// @notice Ordinary public keeper facade, controlled by a distinct local development EOA.
contract AF011KeeperActor {
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant BOOK = 0x81E3b03Cf29865b6c0f85925839567e2Bc61073D;
    address private immutable CONTROLLER;

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

    function updateMarkPrice(uint256 desiredBasketPrice) external onlyController {
        IAF011ReleaseRouter(ROUTER).updateMarkPrice(_priceData(desiredBasketPrice));
    }

    function executeOrder(uint64 orderId, uint256 desiredBasketPrice) external onlyController {
        IAF011ReleaseRouter(ROUTER).executeOrder(orderId, _priceData(desiredBasketPrice));
    }

    function triggerProtection(uint64 protectionId, uint256 desiredBasketPrice)
        external
        onlyController
        returns (uint64 orderId)
    {
        orderId = IPositionProtectionActions(BOOK)
            .triggerPositionProtection(protectionId, _priceData(desiredBasketPrice));
    }

    function expireOrder(uint64 orderId) external onlyController {
        OrderV3Types.ExecutionResult memory result = IAF011ReleaseRouter(ROUTER).expireOrder(orderId);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "expiry did not terminalize");
        require(result.terminalReason == OrderV3Types.TerminalReason.Expired, "wrong terminal reason");
    }

    function _priceData(uint256 desiredBasketPrice) private pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(desiredBasketPrice);
    }
}

/// @notice Unprivileged helper exposing only the preregistered public retry sequences.
contract AF011OutsiderHelper {
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant BOOK = 0x81E3b03Cf29865b6c0f85925839567e2Bc61073D;
    address private immutable CONTROLLER;
    uint64 public lastOrderId;

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

    function retryOnly(uint64 protectionId) external onlyController returns (uint64 orderId) {
        orderId = IPositionProtectionActions(BOOK).retryPositionProtectionClose(protectionId);
        lastOrderId = orderId;
    }

    function expireAndRetry(uint64 expiredOrderId, uint64 protectionId)
        external
        onlyController
        returns (uint64 replacementOrderId)
    {
        OrderV3Types.ExecutionResult memory result = IAF011ReleaseRouter(ROUTER).expireOrder(expiredOrderId);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "expiry did not terminalize");
        require(result.terminalReason == OrderV3Types.TerminalReason.Expired, "wrong terminal reason");
        replacementOrderId = IPositionProtectionActions(BOOK).retryPositionProtectionClose(protectionId);
        lastOrderId = replacementOrderId;
    }
}
```

<!--LocalArbSysAdapter.sol-->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

/// @notice Minimal local-Anvil compatibility adapter for Arbitrum's external ArbSys precompile.
/// @dev Installed only at local address(100). After one empty local activation block, Anvil's
///      block.number follows its local L2-style RPC block sequence. This supplies exactly the
///      method consumed by ExecutionBlockNumberLib and does not replace or write any in-scope
///      Plether contract.
contract LocalArbSysAdapter {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}
```

<!--LocalPythAdapter.sol-->
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";

/// @notice Deterministic local-fork adapter installed only at the external Pyth address on Anvil.
/// @dev The six feed ids, bases, inversion flags, and order are the immutable deployed Plether v2.0.0 basket.
///      The first update blob is abi.encode(desiredBasketPrice). No signed-data claim is made: this contract is an
///      explicitly disclosed local Pyth mock and never replaces or writes an in-scope Plether contract.
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
        require(desiredBasketPrice == 100_000_000 || desiredBasketPrice == 90_000_000, "unregistered local price");

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
        // Four inverse components each floor at basket aggregation. For both pre-registered prices, adding 47
        // normalized units to the 57.6% direct component exactly compensates the aggregate 23-unit floor loss.
        if (index == 0) {
            normalized += 47;
        }
        require(normalized != 0, "zero normalized price");

        // With expo -12, Plether normalizes a direct feed as raw/1e4 and an inverse feed as ~1e20/raw.
        uint256 raw = _inverted(index) ? (1e20 + (normalized / 2)) / normalized : normalized * 1e4;
        require(raw <= uint256(int256(type(int64).max)), "raw price overflow");
        // The explicit bound above proves both casts preserve the positive value.
        // forge-lint: disable-next-line(unsafe-typecast)
        return int64(int256(raw));
    }

    function _requireKnownId(bytes32 id) private pure {
        for (uint256 i; i < 6; ++i) {
            if (id == _feedId(i)) {
                return;
            }
        }
        revert("unknown local feed");
    }

    function _feedId(uint256 index) private pure returns (bytes32) {
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

    function _basePrice(uint256 index) private pure returns (uint256) {
        if (index == 0) {
            return 117_500_000;
        }
        if (index == 1) {
            return 638_000;
        }
        if (index == 2) {
            return 134_480_000;
        }
        if (index == 3) {
            return 72_880_000;
        }
        if (index == 4) {
            return 10_860_000;
        }
        if (index == 5) {
            return 126_100_000;
        }
        revert("base index");
    }

    function _inverted(uint256 index) private pure returns (bool) {
        require(index < 6, "inversion index");
        return index != 0 && index != 2;
    }
}
```


