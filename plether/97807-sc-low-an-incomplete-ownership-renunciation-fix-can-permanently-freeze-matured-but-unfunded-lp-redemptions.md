# \#97807 [SC-Low] An incomplete ownership-renunciation fix can permanently freeze matured but unfunded LP redemptions

**Submitted on Nov 3rd 2026 at 09:59:43 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97807
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Summary

The frozen fix for Cecuro LOW-59 is incomplete. LOW-59 recommended preventing
ownership renunciation in every governance-bearing perps contract. The published
fix added guards to `HousePool`, `OrderRouterAdmin`, and
`EmergencyPauseCoordinator`, but omitted the `CfdTreasury` / `CfdEngine`
ownership lineage even though the Engine owner is the only authority that can
clear the protocol's degraded-mode latch.

The inherited `renounceOwnership()` therefore remains callable on `CfdEngine`.
After Engine ownership becomes the zero address, an ordinary later Router
settlement can latch `degradedMode`. The latch does not clear automatically when
all positions and both directional liabilities reach zero. At the same time,
`HousePool` refuses to fund any Senior or Junior LP redemption epoch while that
latch is set. Because `clearDegradedMode()` is `onlyOwner`, renunciation cannot be
reversed, and the deployment's Engine, Pool, and vault bindings are immutable or
set once, every redemption request that matures without already being funded
remains unable to settle or be cancelled.

Already-funded redemption claims remain claimable and are not included in this
report.

## Prior finding and incomplete-fix disclosure

This report does not claim that the ownership-renunciation family is new.
Cecuro LOW-59, "Owner can leave emergency recovery ownerless," recommended
overriding `renounceOwnership()` in every governance-bearing perps contract. Its
fix commit, `1151d01fd30d5fe6f6d80d604c5ddd4b90845dc6`, added the guard to:

- `packages/perps/src/HousePool.sol:264-267`;
- `packages/perps/src/OrderRouterAdmin.sol:124-127`; and
- `packages/perps/src/EmergencyPauseCoordinator.sol:101-104`.

It did not add the guard to `CfdTreasury` or `CfdEngine`. The Engine already
contained the owner-only degraded-mode recovery transition, so this is the same
root left unfixed in another governance-bearing contract, with a separately
demonstrated permanent LP-funds consequence. LOW-59's own impact text already
names LP epoch settlement becoming permanently unavailable after a later pause;
this report shows the same class of consequence through the Engine's
degraded-mode latch, which the three guards do not cover. I am submitting it
under the program rule that permits a same-root report when the frozen fix is
incomplete.

The report does not rely on a compromised key, a malicious owner, or owner value
extraction. The only owner action in the chain is one call to the inherited
`renounceOwnership()`. The program lists the Owner's immediate powers,
including recapitalizing the pool and clearing degraded mode, but does not list
renunciation among them. The repository documents that "renouncing ownership
does not prevent activation" of a scheduled protocol treasury
(`packages/perps/SECURITY.md:128`) and describes ownership changes as a
two-step transfer (`packages/perps/DEPLOYMENT.md:302`). It does not present
renunciation as an operating procedure, and it keeps renunciation callable on
the Engine.

This report relies on two program provisions. The in-scope provision covers "a
legitimate, documented action by a trusted role that triggers a bug harming
users or the protocol." The privileged-address standard adopted by the program
also preserves a report "where an in-scope impact arises from a logic error that
a privileged address can trigger." Whether one renunciation call qualifies is
an interpretation of those provisions. The known-issues rule that applies to
this report is separate: "For findings marked as fixed or
Acknowledged in the reports above, a report on the same root cause is valid
only if it shows that the fix at commit
a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new
bug." The harmful latch is entered later by ordinary Router settlement and
affects LPs distinct from the owner.

The program excludes attacks that require a trusted role "to act maliciously,
to be compromised, or to act outside its documented operational procedures",
and "Impacts involving centralization risks". This report assumes no malice and
no compromise, and it is not about key loss. It is about a callable code path: the
project already treats an unguarded `renounceOwnership()` as a defect worth a
code fix in three sibling contracts, and the Engine owner holds the only edge
that restores LP redemption funding after the latch.

## Root cause and execution path

1. `CfdEngine` inherits `CfdTreasury`, which inherits `Ownable2Step`:
   `packages/perps/src/CfdEngine.sol:37-46` and
   `packages/perps/src/CfdTreasury.sol:4-10`.
2. Neither contract overrides `renounceOwnership()`. OpenZeppelin's inherited
   function transfers ownership to `address(0)`, while `Ownable2Step` also
   deletes any pending owner:
   `lib/openzeppelin-contracts/contracts/access/Ownable.sol:69-78` and
   `Ownable2Step.sol:48-65`.
3. Ordinary close or liquidation settlement latches `degradedMode` when pool
   assets net of trader claims fall below the remaining maximum-liability
   envelope: `packages/perps/src/CfdEngine.sol:1320-1368`.
4. The sole transition that clears the latch is
   `clearDegradedMode() external onlyOwner`: `CfdEngine.sol:655-667`.
5. LP exits are asynchronous. `HousePool.settleLpEpoch()` must fund a matured
   redemption before the vault claim can pay the LP:
   `packages/perps/src/HousePool.sol:794-883,944-1020` and
   `packages/perps/src/TrancheVault.sol:771-798,998-1020,1148-1212`.
6. `HousePool._requireWithdrawalsLive()` unconditionally reverts with
   `HousePool__DegradedMode()` while the Engine latch is set:
   `HousePool.sol:1864-1870`.
7. Closing every position and reducing both side liabilities to zero does not
   clear the stored latch. With owner and pending owner both zero, no ownership
   transfer or other existing administration path can invoke the only clear
   transition. The Engine dependencies, Pool, terminal book, and Router are set
   once; the Pool's Engine is immutable and its two vault addresses are set once;
   and each vault's Pool is immutable: `CfdEngine.sol:339-440`,
   `HousePool.sol:129-142,269-297`, and `TrancheVault.sol:53-55,251-261`.

## Impact and Critical classification

The affected property is LP access to USDG backing matured, not-yet-funded
Senior and Junior redemption requests. The funds remain in `HousePool`. Once a
request matures, `cancelRedeemRequest()` rejects cancellation, while the degraded
gate prevents settlement from reaching the funding phase
(`TrancheVault.sol:877-914`; `HousePool.sol:794-883`). The LP can therefore
neither recover the escrowed shares nor claim the corresponding assets through
any path provided by the frozen contracts.

The local-fork PoC proves the terminal conditions rather than inferring them:

- the deployed Engine owner becomes zero in the renounced branch;
- an ordinary close executed through the deployed Router causes the Engine to
  latch degraded mode;
- all LONG and SHORT positions are then closed;
- both aggregate side liabilities are zero;
- degraded mode remains latched;
- a real matured Junior redemption stays pending and has zero claimable amount;
- settlement reverts with exact selector `0x9f7adff3`,
  `HousePool__DegradedMode()`;
- recovery reverts with exact selector `0x118cdaa7`,
  `OwnableUnauthorizedAccount(address)`; and
- the same state from the same pre-branch snapshot settles and pays the same LP
  when ownership remains intact and the owner clears the latch.

This matches the listed Critical impact "Permanent freezing of funds." The PoC
uses a 1-USDG Junior deposit as a nonzero witness. The position sizes are
chosen so that, with the pool as it stood at the pinned block, the stressed close
leaves effective assets below the maximum liability: the index falls from 1.00
to 0.20, an 80 percent decline, and a deeper pool would need larger positions or
a larger move to latch. Senior funding passes through the same degraded-mode
check in the same settlement function. I make no claim about production TVL, a
total affected amount, a particular market move, pool utilization, or
frequency.

## Recommended remediation

Override `renounceOwnership()` in the Engine ownership lineage and revert,
matching the three existing guards. For example, place the override in
`CfdTreasury` so every Engine instance inherits it:

```solidity
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

Also:

1. update the repository assumptions and the sentence in `SECURITY.md` that
   assume, at the frozen commit, that Engine renunciation succeeds;
2. add automated regression coverage that composes renunciation, ordinary
   Router settlement into degraded mode, complete position/liability cleanup,
   and a matured LP redemption;
3. assert that every contract holding a unique recovery authority rejects
   irreversible ownership loss; and
4. consider a separately governed recovery design for the degraded latch so one
   ownership transition cannot permanently remove the only LP-exit recovery
   edge.

The narrow guard is sufficient to stop the demonstrated chain at its first
step. On the local fork, a standalone control contract that carries the same
one-function guard rejects the renunciation call: the transaction reverts and
the owner is retained. The control is not a patched Engine, and it reverts with
its own error name; the remediation above reverts with
`OwnableInvalidOwner(address(0))`, as the three existing guards do. With the
first step rejected, the ownerless terminal cannot be created.

## Scope of the claim

The PoC proves the complete deployed-code path on the pinned local fork. It does
not claim that ownership has already been renounced on a public deployment, that
the measured test deployment holds production value, or that a particular
market move will occur. Those questions are separate from the demonstrated
frozen-source defect and its terminal consequence.

The fork uses the deployed release's own economics, which match the release
parameters the program page states: maintenance margin 0.1%, initial margin
0.2%, execution fee 0.04%, frozen-close spread 0.50%, and a Senior cap of 40M
USDG at 80% of the pool. The program states that the frozen code is intended for
Arbitrum One with USDG and lists the v2.0.0 Arbitrum Sepolia deployment as the
fork-testing resource; the release identifies six-decimal Mock USDG and test
liquidity. This PoC therefore establishes deployed-code and release-shape
behavior, not a live production state. Pool size matters to whether and when the
latch occurs, and no production pool size is assumed.

## References

- [Current Plether competition scope and PoC rules](https://immunefi.com/audit-competition/plether/scope/)
- [Cecuro audit containing LOW-59](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [`CfdEngine`: owner-only clear and degraded-mode latch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L667)
- [`HousePool`: degraded-mode gate before redemption funding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L883)
- [`TrancheVault`: matured redemption cancellation is unavailable](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L877-L914)
- [v2.0.0 Arbitrum Sepolia release](https://github.com/Plether-Fi/plether-core/releases/tag/v2.0.0)


## Link to Proof of Concept

https://gist.github.com/AFestive/70f616da2e2afccc7cf6b0096ec46b27

## Proof of Concept

## Proof of concept

### Safety, identity, and code delivery

The seven complete target/harness source files are inline below. Python
comments and docstrings were removed mechanically to fit Immunefi's 65,535
character field limit; each normalized AST matches the corresponding retained
105/105-PASS source. The four Solidity files are byte-identical to the retained
run. No exploit logic is delegated to the Gist.

The secret Gist contains only delivery/support material: the restricted
state-read proxy, portable path/build wrapper, Foundry configuration,
preregistration, release/code-hash checks, and retained logs/results. It also
retains the original-format archive for byte-level evidence verification. The
inline files are the PoC; the Gist is supporting evidence and safe execution
plumbing.

This runs only on a fresh local Anvil fork of Arbitrum Sepolia v2.0.0 at block
`323664360`, chain ID `421614`, from commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Before local adapters are
installed, the wrapper checks all 30 deployed runtime hashes. It modifies only
the external Pyth and `ArbSys` addresses on local Anvil, never an in-scope
runtime, and sends no public-chain transaction.

### Reproduction

1. Clone the repository recursively, detach at the commit above, and install
   the pinned Foundry 1.5.1 toolchain plus Solc 0.8.35.
2. Restore the seven inline files at their headings. Download the supporting
   files from secret Gist `70f616da2e2afccc7cf6b0096ec46b27` and place them
   beside `executed/` as described by its README.
3. Supply an Arbitrum Sepolia archive endpoint through
   `PLETHER_FORK_RPC_URL`, then run once with a new evidence path:

```bash
python3 portable_run.py \
  --repo /path/to/audit-comp-plether \
  --foundry-bin /path/to/foundry-1.5.1/bin \
  --evidence-dir /path/to/new-empty-parent/af001-run-01 \
  --ack-final-run
```

The retained run produced 105 PASS rows, no FAIL row, and 33 local
transactions. The decisive output includes:

```text
renounced-owner-is-zero: PASS
renounced-owner-branch-long-liability-terminal: PASS
renounced-owner-branch-short-liability-terminal: PASS
renounced-owner-branch-degraded-still-latched: PASS
renounced-owner-branch-redemption-remains-unfunded: PASS
owner-intact-branch-degraded-cleared: PASS
owner-intact-branch-lp-payment: PASS
renunciation-guard-control: PASS
run-result: PASS
```

The ownerless branch closes both side liabilities to zero but leaves
`degradedMode` latched, so the matured LP redemption remains pending with zero
claimable amount. Reverting to the identical snapshot while retaining the
owner lets `clearDegradedMode()` unlock the same redemption and pays the LP
`0.680312` USDG. The guard control rejects renunciation and retains ownership.

### `executed/run_once.py`

```python
from __future__ import annotations
import argparse
import hashlib
import json
import pathlib
import socket
import subprocess
import sys
import time
import urllib.error
import urllib.request
ROOT = pathlib.Path(__file__).resolve().parents[4]
PLETHER = ROOT / 'plether-engagement-2026-11'
CODE = pathlib.Path(__file__).resolve().parent
SOURCE = PLETHER / 'source' / 'audit-comp-plether'
MANIFEST = SOURCE / 'deployments' / 'releases' / '2026-10-28-perps-arbitrum-sepolia-v2.0.0' / 'manifest.json'
OUT = pathlib.Path('/data/market/debug-contracts/scratch/plether-af001-local-fork-2026-11-02-codex-r01/code/out')
LOG_DIR = pathlib.Path('/data/logs/plether-af001-local-fork-2026-11-02-codex-r01')
FOUNDRY = PLETHER / '.scratch' / 'toolchains' / 'foundry-v1.5.1' / 'versions' / 'foundry-rs' / 'foundry' / 'v1.5.1'
CAST = FOUNDRY / 'cast'
ANVIL = FOUNDRY / 'anvil'
PROXY = ROOT / 'docs' / 'venture' / 'tools' / 'rpc_key_proxy.py'
PROXY_PORT = 28680
ANVIL_PORT = 28681
PROXY_LOCAL_URL = 'http://127.0.0.1:28680'
ANVIL_LOCAL_URL = 'http://127.0.0.1:28681'
UPSTREAM_HOST = 'arb-sepolia.g.alchemy.com'
FORK_BLOCK = 323664360
CHAIN_ID = 421614
TOKEN = '0x3d051dbbf5ff427a4ae8ad09746902666af8fb9d'
ENGINE = '0xca18a6783d7542f9d2f28f4f259772b74c4d6baa'
POOL = '0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e'
VAULT = '0x4a8b67184a1e3224c11a191cc69f22c50eef734e'
PYTH = '0x0b73614636c855bf23f342f307fb981a3e47f42b'
OWNER = '0x5a71a4094ec81165ada48aa4c27da48ec27e0d6b'
PRICE_ENTRY = 100000000
PRICE_STRESS = 20000000
LP_ASSETS = 100000 * 10 ** 6
LONG_FUNDS = 100000 * 10 ** 6
SHORT_FUNDS = 200000 * 10 ** 6
SIZE_400K = 400000 * 10 ** 18
SIZE_300K = 300000 * 10 ** 18
SIZE_700K = 700000 * 10 ** 18
MARGIN_50K = 50000 * 10 ** 6
RUN_LOG = LOG_DIR / 'run-final.jsonl'
RESULT = LOG_DIR / 'result-final.json'
PROXY_AUDIT = LOG_DIR / 'proxy-final.jsonl'
PROXY_STDOUT = LOG_DIR / 'proxy-final.stdout.log'
ANVIL_STDOUT = LOG_DIR / 'anvil-final.log'
_run_handle = None
_tx_count = 0

class RpcFailure(RuntimeError):
    pass

def _exclusive_paths() -> list[pathlib.Path]:
    return [RUN_LOG, RESULT, PROXY_AUDIT, PROXY_STDOUT, ANVIL_STDOUT]

def record(step: str, status: str, **details) -> None:
    row = {'time': time.strftime('%Y-%m-%dT%H:%M:%S%z'), 'step': step, 'status': status, **details}
    assert _run_handle is not None
    _run_handle.write(json.dumps(row, sort_keys=True) + '\n')
    _run_handle.flush()
    print(f'{step}: {status}', flush=True)

def rpc(method: str, params: list, local_url: str=ANVIL_LOCAL_URL):
    payload = json.dumps({'jsonrpc': '2.0', 'id': 1, 'method': method, 'params': params}).encode()
    request = urllib.request.Request(local_url, data=payload, headers={'Content-Type': 'application/json'})
    try:
        with urllib.request.urlopen(request, timeout=120) as response:
            body = json.loads(response.read())
    except urllib.error.HTTPError as error:
        raw = error.read()
        try:
            body = json.loads(raw)
        except json.JSONDecodeError as decode_error:
            raise RpcFailure(f'{method} HTTP {error.code} non-JSON response') from decode_error
    if 'error' in body:
        error = body['error']
        raise RpcFailure(f"{method}: code={error.get('code')} message={error.get('message')} data={error.get('data')}")
    return body['result']

def wait_for_anvil(process: subprocess.Popen) -> None:
    deadline = time.monotonic() + 180
    last_error = 'not attempted'
    while time.monotonic() < deadline:
        if process.poll() is not None:
            raise RuntimeError(f'Anvil exited during startup with {process.returncode}')
        try:
            if int(rpc('eth_chainId', []), 16) == CHAIN_ID:
                return
        except Exception as error:
            last_error = type(error).__name__
        time.sleep(0.25)
    raise RuntimeError(f'Anvil startup timeout; last error type {last_error}')

def ensure_port_free(port: int) -> None:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.settimeout(0.2)
        if sock.connect_ex(('127.0.0.1', port)) == 0:
            raise RuntimeError(f'loopback port {port} is already in use')

def start_processes():
    ensure_port_free(PROXY_PORT)
    ensure_port_free(ANVIL_PORT)
    proxy_stdout = open(PROXY_STDOUT, 'x', encoding='utf-8')
    anvil_stdout = open(ANVIL_STDOUT, 'x', encoding='utf-8')
    proxy = None
    anvil = None
    try:
        proxy = subprocess.Popen([sys.executable, str(PROXY), '--port', str(PROXY_PORT), '--host', UPSTREAM_HOST, '--log', str(PROXY_AUDIT), '--max-requests', '100000'], cwd=ROOT, stdout=proxy_stdout, stderr=subprocess.STDOUT, text=True)
        time.sleep(0.25)
        if proxy.poll() is not None:
            raise RuntimeError(f'proxy exited during startup with {proxy.returncode}')
        anvil = subprocess.Popen([str(ANVIL), '--quiet', '--host', '127.0.0.1', '--port', str(ANVIL_PORT), '--fork-url', PROXY_LOCAL_URL, '--fork-block-number', str(FORK_BLOCK), '--no-rate-limit'], cwd=ROOT, stdout=anvil_stdout, stderr=subprocess.STDOUT, text=True)
        record('process-start', 'PASS', proxy_pid=proxy.pid, anvil_pid=anvil.pid)
        wait_for_anvil(anvil)
        return (proxy, anvil, proxy_stdout, anvil_stdout)
    except Exception:
        for process in (anvil, proxy):
            if process is not None and process.poll() is None:
                process.terminate()
                try:
                    process.wait(timeout=10)
                except subprocess.TimeoutExpired:
                    process.kill()
                    process.wait(timeout=10)
        anvil_stdout.close()
        proxy_stdout.close()
        raise

def stop_process(name: str, process: subprocess.Popen | None) -> None:
    if process is None:
        return
    if process.poll() is None:
        process.terminate()
        try:
            process.wait(timeout=10)
        except subprocess.TimeoutExpired:
            process.kill()
            process.wait(timeout=10)
    record(f'process-stop-{name}', 'PASS', pid=process.pid, exit_code=process.returncode)

def cast(*args: str) -> str:
    return subprocess.check_output([str(CAST), *args], cwd=ROOT, text=True).strip()

def calldata(signature: str, *args) -> str:
    return cast('calldata', signature, *(str(value) for value in args))

def selector(signature: str) -> str:
    return cast('sig', signature).lower()

def keccak_hex(value: str) -> str:
    return cast('keccak', value).lower()

def artifact(contract_file: str, contract_name: str) -> dict:
    path = OUT / contract_file / f'{contract_name}.json'
    with open(path, encoding='utf-8') as handle:
        return json.load(handle)

def object_hex(value: str) -> str:
    return value if value.startswith('0x') else '0x' + value

def wait_receipt(tx_hash: str) -> dict:
    deadline = time.monotonic() + 120
    while time.monotonic() < deadline:
        receipt = rpc('eth_getTransactionReceipt', [tx_hash])
        if receipt is not None:
            return receipt
        time.sleep(0.05)
    raise RuntimeError(f'receipt timeout for {tx_hash}')

def send(sender: str, to: str | None, data: str, label: str, expected_status: int=1) -> dict:
    global _tx_count
    transaction = {'from': sender, 'data': data, 'gas': hex(100000000), 'value': '0x0'}
    if to is not None:
        transaction['to'] = to
    tx_hash = rpc('eth_sendTransaction', [transaction])
    receipt = wait_receipt(tx_hash)
    status = int(receipt['status'], 16)
    _tx_count += 1
    record(label, 'PASS' if status == expected_status else 'FAIL', tx_hash=tx_hash, receipt=receipt, expected_status=expected_status)
    if status != expected_status:
        raise AssertionError(f'{label}: transaction status {status}, expected {expected_status}')
    return receipt

def deploy(sender: str, contract_file: str, contract_name: str, label: str) -> str:
    build = artifact(contract_file, contract_name)
    receipt = send(sender, None, object_hex(build['bytecode']['object']), label)
    address = receipt.get('contractAddress')
    if not address:
        raise AssertionError(f'{label}: no contract address')
    return address.lower()

def call_raw(to: str, signature: str, *args) -> str:
    return rpc('eth_call', [{'to': to, 'data': calldata(signature, *args)}, 'latest'])

def words(raw: str) -> list[int]:
    payload = raw[2:]
    if len(payload) % 64:
        raise AssertionError('non-word-aligned ABI response')
    return [int(payload[offset:offset + 64], 16) for offset in range(0, len(payload), 64)]

def uint_call(to: str, signature: str, *args) -> int:
    values = words(call_raw(to, signature, *args))
    if len(values) != 1:
        raise AssertionError(f'{signature}: expected one word, got {len(values)}')
    return values[0]

def bool_call(to: str, signature: str, *args) -> bool:
    value = uint_call(to, signature, *args)
    if value not in (0, 1):
        raise AssertionError(f'{signature}: invalid bool {value}')
    return bool(value)

def address_call(to: str, signature: str, *args) -> str:
    raw = call_raw(to, signature, *args)
    if len(raw) != 66:
        raise AssertionError(f'{signature}: invalid address ABI length')
    return '0x' + raw[-40:].lower()

def bytes4_call(to: str, signature: str, *args) -> str:
    raw = call_raw(to, signature, *args)
    if len(raw) != 66:
        raise AssertionError(f'{signature}: invalid bytes4 ABI length')
    return ('0x' + raw[2:10]).lower()

def current_timestamp() -> int:
    block = rpc('eth_getBlockByNumber', ['latest', False])
    return int(block['timestamp'], 16)

def advance_to(timestamp: int, label: str) -> None:
    current = current_timestamp()
    if timestamp > current:
        rpc('evm_setNextBlockTimestamp', [timestamp])
    rpc('evm_mine', [])
    observed = current_timestamp()
    if observed < timestamp:
        raise AssertionError(f'{label}: timestamp {observed} below {timestamp}')
    record(label, 'PASS', requested=timestamp, observed=observed)

def advance(seconds: int, label: str) -> None:
    rpc('evm_increaseTime', [seconds])
    rpc('evm_mine', [])
    record(label, 'PASS', seconds=seconds, observed=current_timestamp())

def assert_equal(label: str, observed, expected) -> None:
    ok = observed == expected
    record(label, 'PASS' if ok else 'FAIL', observed=observed, expected=expected)
    if not ok:
        raise AssertionError(f'{label}: observed {observed!r}, expected {expected!r}')

def assert_positive(label: str, observed: int) -> None:
    ok = observed > 0
    record(label, 'PASS' if ok else 'FAIL', observed=observed)
    if not ok:
        raise AssertionError(f'{label}: expected positive value')

def manifest_runtime_check() -> None:
    with open(MANIFEST, encoding='utf-8') as handle:
        release = json.load(handle)
    rows = []
    for name, item in release['contracts'].items():
        address = item['address'].lower()
        code = rpc('eth_getCode', [address, 'latest'])
        actual = keccak_hex(code)
        expected = item['runtimeCodeHash'].lower()
        rows.append({'name': name, 'address': address, 'actual': actual, 'expected': expected})
        if actual != expected:
            record('runtime-hashes', 'FAIL', rows=rows)
            raise AssertionError(f'runtime mismatch: {name}')
    record('runtime-hashes', 'PASS', count=len(rows), rows=rows)

def set_external_pyth_adapter() -> None:
    build = artifact('LocalPythAdapter.sol', 'LocalPythAdapter')
    runtime = object_hex(build['deployedBytecode']['object'])
    rpc('anvil_setCode', [PYTH, runtime])
    observed = rpc('eth_getCode', [PYTH, 'latest'])
    assert_equal('external-pyth-adapter-code', keccak_hex(observed), keccak_hex(runtime))

def actor_send(sender: str, actor: str, signature: str, args: tuple, label: str) -> dict:
    return send(sender, actor, calldata(signature, *args), label)

def commit_execute(sender: str, actor: str, side: int, close: bool, size: int, margin: int, price: int, label: str) -> int:
    if close:
        actor_send(sender, actor, 'commitClose(uint8,uint256)', (side, size), f'{label}-commit')
    else:
        actor_send(sender, actor, 'commitOpen(uint8,uint256,uint256)', (side, size, margin), f'{label}-commit')
    order_id = uint_call(actor, 'lastOrderId()')
    advance(2, f'{label}-post-commit-time')
    actor_send(sender, actor, 'executeLast(uint256)', (price,), f'{label}-execute')
    record(f'{label}-order', 'PASS', order_id=order_id, actor=actor, price=price)
    return order_id

def position_size(account: str) -> int:
    return words(call_raw(ENGINE, 'positions(address)', account))[0]

def side_liability(side: int) -> int:
    return words(call_raw(ENGINE, 'sides(uint256)', side))[0]

def terminal_sequence(sender: str, long_actor: str, short_actor: str, branch: str) -> None:
    commit_execute(sender, short_actor, 1, False, SIZE_400K, MARGIN_50K, PRICE_ENTRY, f'{branch}-short-open-400k')
    commit_execute(sender, long_actor, 0, False, SIZE_400K, MARGIN_50K, PRICE_ENTRY, f'{branch}-long-open-400k')
    commit_execute(sender, short_actor, 1, False, SIZE_300K, MARGIN_50K, PRICE_ENTRY, f'{branch}-short-open-300k')
    assert_equal(f'{branch}-long-size-open', position_size(long_actor), SIZE_400K)
    assert_equal(f'{branch}-short-size-open', position_size(short_actor), SIZE_700K)
    commit_execute(sender, long_actor, 0, True, SIZE_400K, 0, PRICE_STRESS, f'{branch}-long-close-stress')
    assert_equal(f'{branch}-degraded-latched', bool_call(ENGINE, 'degradedMode()'), True)
    commit_execute(sender, short_actor, 1, True, SIZE_700K, 0, PRICE_ENTRY, f'{branch}-short-close-terminal')
    assert_equal(f'{branch}-long-size-terminal', position_size(long_actor), 0)
    assert_equal(f'{branch}-short-size-terminal', position_size(short_actor), 0)
    assert_equal(f'{branch}-long-liability-terminal', side_liability(0), 0)
    assert_equal(f'{branch}-short-liability-terminal', side_liability(1), 0)
    assert_equal(f'{branch}-degraded-still-latched', bool_call(ENGINE, 'degradedMode()'), True)

def request_mature_redeem(sender: str, lp_actor: str, branch: str) -> int:
    actor_send(sender, lp_actor, 'requestAllJuniorRedeem()', (), f'{branch}-request-redeem')
    request_id = uint_call(lp_actor, 'lastRedeemId()')
    epoch_start = uint_call(POOL, 'lpEpochStart(uint256)', request_id)
    advance_to(epoch_start + 1, f'{branch}-redeem-maturity')
    pending = uint_call(VAULT, 'pendingRedeemRequest(uint256,address)', request_id, lp_actor)
    assert_positive(f'{branch}-pending-before-settlement', pending)
    return request_id

def probe_blocked_settlement(sender: str, lp_actor: str, request_id: int, branch: str) -> None:
    actor_send(sender, lp_actor, 'probeSettleLpEpoch(uint256)', (PRICE_ENTRY,), f'{branch}-probe-settlement')
    assert_equal(f'{branch}-settlement-call-failed', bool_call(lp_actor, 'lastProbeOk()'), False)
    assert_equal(f'{branch}-settlement-selector', bytes4_call(lp_actor, 'lastProbeSelector()'), selector('HousePool__DegradedMode()'))
    pending = uint_call(VAULT, 'pendingRedeemRequest(uint256,address)', request_id, lp_actor)
    claimable = uint_call(VAULT, 'claimableRedeemRequest(uint256,address)', request_id, lp_actor)
    assert_positive(f'{branch}-redemption-remains-pending', pending)
    assert_equal(f'{branch}-redemption-remains-unfunded', claimable, 0)

def impersonated_owner_send(data: str, label: str) -> None:
    rpc('anvil_setBalance', [OWNER, hex(10 ** 20)])
    rpc('anvil_impersonateAccount', [OWNER])
    try:
        send(OWNER, ENGINE, data, label)
    finally:
        rpc('anvil_stopImpersonatingAccount', [OWNER])

def file_sha256(path: pathlib.Path) -> str:
    digest = hashlib.sha256()
    with open(path, 'rb') as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b''):
            digest.update(chunk)
    return digest.hexdigest()

def main() -> int:
    global _run_handle
    parser = argparse.ArgumentParser()
    parser.add_argument('--ack-final-run', action='store_true', help='acknowledge consumption of the one pre-registered local target run')
    args = parser.parse_args()
    if not args.ack_final_run:
        parser.error('--ack-final-run is required')
    LOG_DIR.mkdir(parents=True, exist_ok=True)
    collisions = [str(path) for path in _exclusive_paths() if path.exists()]
    if collisions:
        raise SystemExit('final evidence path already exists: ' + ', '.join(collisions))
    _run_handle = open(RUN_LOG, 'x', encoding='utf-8')
    proxy = None
    anvil = None
    proxy_stdout = None
    anvil_stdout = None
    result = {'result': 'FAIL', 'fork_block': FORK_BLOCK, 'chain_id': CHAIN_ID}
    try:
        record('run-start', 'PASS', argv=[pathlib.Path(sys.argv[0]).name, '--ack-final-run'], python=sys.version, cast_version=cast('--version'), anvil_version=subprocess.check_output([str(ANVIL), '--version'], text=True).strip(), source_hashes={path.name: file_sha256(path) for path in sorted((CODE / 'src').glob('*.sol'))} | {'run_once.py': file_sha256(CODE / 'run_once.py'), 'PRE_REGISTERED_V2.md': file_sha256(CODE.parent / 'PRE_REGISTERED_V2.md'), 'release-manifest.json': file_sha256(MANIFEST)})
        proxy, anvil, proxy_stdout, anvil_stdout = start_processes()
        assert_equal('chain-id', int(rpc('eth_chainId', []), 16), CHAIN_ID)
        assert_equal('fork-block', int(rpc('eth_blockNumber', []), 16), FORK_BLOCK)
        manifest_runtime_check()
        assert_equal('engine-owner-precondition', address_call(ENGINE, 'owner()'), OWNER)
        assert_equal('engine-degraded-precondition', bool_call(ENGINE, 'degradedMode()'), False)
        set_external_pyth_adapter()
        accounts = [account.lower() for account in rpc('eth_accounts', [])]
        if not accounts:
            raise AssertionError('Anvil returned no unlocked account')
        sender = accounts[0]
        lp_actor = deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-lp-actor')
        long_actor = deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-long-actor')
        short_actor = deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-short-actor')
        guard = deploy(sender, 'RenounceGuardControl.sol', 'RenounceGuardControl', 'deploy-guard-control')
        record('harness-addresses', 'PASS', sender=sender, lp_actor=lp_actor, long_actor=long_actor, short_actor=short_actor, guard=guard)
        actor_send(sender, lp_actor, 'fundAndRequestJuniorDeposit(uint256)', (LP_ASSETS,), 'lp-request-deposit')
        deposit_id = uint_call(lp_actor, 'lastDepositId()')
        deposit_start = uint_call(VAULT, 'depositEpochStart(uint256)', deposit_id)
        advance_to(deposit_start, 'lp-deposit-maturity')
        actor_send(sender, lp_actor, 'settleLpEpoch(uint256)', (PRICE_ENTRY,), 'lp-deposit-settlement')
        claimable_assets = uint_call(VAULT, 'claimableDepositRequest(uint256,address)', deposit_id, lp_actor)
        assert_equal('lp-deposit-claimable-assets', claimable_assets, LP_ASSETS)
        actor_send(sender, lp_actor, 'claimJuniorDepositAll()', (), 'lp-claim-deposit')
        assert_positive('lp-victim-shares', uint_call(lp_actor, 'juniorShares()'))
        advance(3601, 'lp-deposit-cooldown')
        actor_send(sender, long_actor, 'fundTrader(uint256)', (LONG_FUNDS,), 'fund-long-trader')
        actor_send(sender, short_actor, 'fundTrader(uint256)', (SHORT_FUNDS,), 'fund-short-trader')
        snapshot = rpc('evm_snapshot', [])
        record('pre-renunciation-snapshot', 'PASS', snapshot=snapshot)
        impersonated_owner_send(calldata('renounceOwnership()'), 'renounced-owner-branch-renounce')
        assert_equal('renounced-owner-is-zero', address_call(ENGINE, 'owner()'), '0x' + '0' * 40)
        terminal_sequence(sender, long_actor, short_actor, 'renounced-owner-branch')
        renounced_redeem = request_mature_redeem(sender, lp_actor, 'renounced-owner-branch')
        probe_blocked_settlement(sender, lp_actor, renounced_redeem, 'renounced-owner-branch')
        actor_send(sender, lp_actor, 'probeClearDegradedMode()', (), 'renounced-owner-branch-probe-clear')
        assert_equal('renounced-owner-branch-clear-call-failed', bool_call(lp_actor, 'lastProbeOk()'), False)
        assert_equal('renounced-owner-branch-clear-selector', bytes4_call(lp_actor, 'lastProbeSelector()'), selector('OwnableUnauthorizedAccount(address)'))
        assert_equal('snapshot-revert', rpc('evm_revert', [snapshot]), True)
        assert_equal('owner-intact-restored-owner', address_call(ENGINE, 'owner()'), OWNER)
        assert_equal('owner-intact-restored-degraded', bool_call(ENGINE, 'degradedMode()'), False)
        terminal_sequence(sender, long_actor, short_actor, 'owner-intact-branch')
        intact_redeem = request_mature_redeem(sender, lp_actor, 'owner-intact-branch')
        probe_blocked_settlement(sender, lp_actor, intact_redeem, 'owner-intact-branch')
        impersonated_owner_send(calldata('clearDegradedMode()'), 'owner-intact-branch-clear')
        assert_equal('owner-intact-branch-degraded-cleared', bool_call(ENGINE, 'degradedMode()'), False)
        actor_send(sender, lp_actor, 'settleLpEpoch(uint256)', (PRICE_ENTRY,), 'owner-intact-branch-settle')
        claimable_shares = uint_call(VAULT, 'claimableRedeemRequest(uint256,address)', intact_redeem, lp_actor)
        assert_positive('owner-intact-branch-claimable-shares', claimable_shares)
        balance_before = uint_call(lp_actor, 'tokenBalance()')
        actor_send(sender, lp_actor, 'claimJuniorRedeemAll()', (), 'owner-intact-branch-claim')
        balance_after = uint_call(lp_actor, 'tokenBalance()')
        assert_positive('owner-intact-branch-lp-payment', balance_after - balance_before)
        send(sender, guard, calldata('renounceOwnership()'), 'renunciation-guard-control', expected_status=0)
        assert_equal('renunciation-guard-owner-retained', address_call(guard, 'owner()'), sender)
        result = {'result': 'PASS', 'chain_id': CHAIN_ID, 'fork_block': FORK_BLOCK, 'transactions': _tx_count, 'lp_actor': lp_actor, 'long_actor': long_actor, 'short_actor': short_actor, 'guard_control': guard}
        record('run-result', 'PASS', **result)
        with open(RESULT, 'x', encoding='utf-8') as handle:
            json.dump(result, handle, indent=2, sort_keys=True)
            handle.write('\n')
        return 0
    except Exception as error:
        result['error_type'] = type(error).__name__
        result['error'] = str(error)
        record('run-result', 'FAIL', **result)
        raise
    finally:
        stop_process('anvil', anvil)
        stop_process('proxy', proxy)
        if anvil_stdout is not None:
            anvil_stdout.close()
        if proxy_stdout is not None:
            proxy_stdout.close()
        if _run_handle is not None:
            _run_handle.close()
if __name__ == '__main__':
    sys.exit(main())
```

### `executed/run_once_v4.py`

```python
from __future__ import annotations
import argparse
import importlib.util
import json
import pathlib
import subprocess
import sys
CODE = pathlib.Path(__file__).resolve().parent
BASE_PATH = CODE / 'run_once.py'
LOG_DIR = pathlib.Path('/data/logs/plether-af001-local-fork-2026-11-02-codex-r03')
LOCAL_TX_GAS = 500000000
ARB_SYS = '0x0000000000000000000000000000000000000064'

def _load_base():
    spec = importlib.util.spec_from_file_location('af001_frozen_v2_runner', BASE_PATH)
    if spec is None or spec.loader is None:
        raise RuntimeError('unable to load frozen V2 runner')
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module
base = _load_base()
base.LOG_DIR = LOG_DIR
base.RUN_LOG = LOG_DIR / 'run-final.jsonl'
base.RESULT = LOG_DIR / 'result-final.json'
base.PROXY_AUDIT = LOG_DIR / 'proxy-final.jsonl'
base.PROXY_STDOUT = LOG_DIR / 'proxy-final.stdout.log'
base.ANVIL_STDOUT = LOG_DIR / 'anvil-final.log'

def send(sender: str, to: str | None, data: str, label: str, expected_status: int=1) -> dict:
    transaction = {'from': sender, 'data': data, 'gas': hex(LOCAL_TX_GAS), 'value': '0x0'}
    if to is not None:
        transaction['to'] = to
    tx_hash = base.rpc('eth_sendTransaction', [transaction], local_url='http://127.0.0.1:28681')
    receipt = base.wait_receipt(tx_hash)
    status = int(receipt['status'], 16)
    base._tx_count += 1
    base.record(label, 'PASS' if status == expected_status else 'FAIL', tx_hash=tx_hash, receipt=receipt, expected_status=expected_status, local_gas_ceiling=LOCAL_TX_GAS)
    if status != expected_status:
        raise AssertionError(f'{label}: transaction status {status}, expected {expected_status}')
    return receipt
base.send = send

def set_external_adapters() -> None:
    base.set_external_pyth_adapter()
    build = base.artifact('LocalArbSysAdapter.sol', 'LocalArbSysAdapter')
    runtime = base.object_hex(build['deployedBytecode']['object'])
    base.rpc('anvil_setCode', [ARB_SYS, runtime], local_url='http://127.0.0.1:28681')
    observed = base.rpc('eth_getCode', [ARB_SYS, 'latest'], local_url='http://127.0.0.1:28681')
    base.assert_equal('external-arbsys-adapter-code', base.keccak_hex(observed), base.keccak_hex(runtime))
    base.rpc('evm_mine', [], local_url='http://127.0.0.1:28681')
    local_block = int(base.rpc('eth_blockNumber', [], local_url='http://127.0.0.1:28681'), 16)
    base.record('external-arbsys-local-activation-block', 'PASS', block_number=local_block)
    base.assert_equal('external-arbsys-block-number', base.uint_call(ARB_SYS, 'arbBlockNumber()'), local_block)

def open_positions(sender: str, long_actor: str, short_actor: str) -> None:
    base.commit_execute(sender, short_actor, 1, False, base.SIZE_400K, base.MARGIN_50K, base.PRICE_ENTRY, 'setup-short-open-400k')
    base.commit_execute(sender, long_actor, 0, False, base.SIZE_400K, base.MARGIN_50K, base.PRICE_ENTRY, 'setup-long-open-400k')
    base.commit_execute(sender, short_actor, 1, False, base.SIZE_300K, base.MARGIN_50K, base.PRICE_ENTRY, 'setup-short-open-300k')
    base.assert_equal('setup-long-size-open', base.position_size(long_actor), base.SIZE_400K)
    base.assert_equal('setup-short-size-open', base.position_size(short_actor), base.SIZE_700K)
    base.assert_equal('setup-degraded-false', base.bool_call(base.ENGINE, 'degradedMode()'), False)

def close_positions(sender: str, long_actor: str, short_actor: str, branch: str) -> None:
    base.commit_execute(sender, long_actor, 0, True, base.SIZE_400K, 0, base.PRICE_STRESS, f'{branch}-long-close-stress')
    base.assert_equal(f'{branch}-degraded-latched', base.bool_call(base.ENGINE, 'degradedMode()'), True)
    base.commit_execute(sender, short_actor, 1, True, base.SIZE_700K, 0, base.PRICE_ENTRY, f'{branch}-short-close-terminal')
    base.assert_equal(f'{branch}-long-size-terminal', base.position_size(long_actor), 0)
    base.assert_equal(f'{branch}-short-size-terminal', base.position_size(short_actor), 0)
    base.assert_equal(f'{branch}-long-liability-terminal', base.side_liability(0), 0)
    base.assert_equal(f'{branch}-short-liability-terminal', base.side_liability(1), 0)
    base.assert_equal(f'{branch}-degraded-still-latched', base.bool_call(base.ENGINE, 'degradedMode()'), True)

def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument('--ack-final-run', action='store_true')
    args = parser.parse_args()
    if not args.ack_final_run:
        parser.error('--ack-final-run is required')
    LOG_DIR.mkdir(parents=True, exist_ok=True)
    collisions = [str(path) for path in base._exclusive_paths() if path.exists()]
    if collisions:
        raise SystemExit('final evidence path already exists: ' + ', '.join(collisions))
    base._tx_count = 0
    base._run_handle = open(base.RUN_LOG, 'x', encoding='utf-8')
    proxy = None
    anvil = None
    proxy_stdout = None
    anvil_stdout = None
    result = {'result': 'FAIL', 'fork_block': base.FORK_BLOCK, 'chain_id': base.CHAIN_ID}
    try:
        base.record('run-start', 'PASS', argv=[pathlib.Path(sys.argv[0]).name, '--ack-final-run'], python=sys.version, cast_version=base.cast('--version'), anvil_version=subprocess.check_output([str(base.ANVIL), '--version'], text=True).strip(), local_tx_gas_ceiling=LOCAL_TX_GAS, source_hashes={path.name: base.file_sha256(path) for path in sorted((CODE / 'src').glob('*.sol'))} | {'run_once.py': base.file_sha256(BASE_PATH), 'run_once_v4.py': base.file_sha256(pathlib.Path(__file__)), 'PRE_REGISTERED_V4.md': base.file_sha256(CODE.parent / 'PRE_REGISTERED_V4.md'), 'release-manifest.json': base.file_sha256(base.MANIFEST)})
        proxy, anvil, proxy_stdout, anvil_stdout = base.start_processes()
        base.assert_equal('chain-id', int(base.rpc('eth_chainId', []), 16), base.CHAIN_ID)
        base.assert_equal('fork-block', int(base.rpc('eth_blockNumber', []), 16), base.FORK_BLOCK)
        base.manifest_runtime_check()
        base.assert_equal('engine-owner-precondition', base.address_call(base.ENGINE, 'owner()'), base.OWNER)
        base.assert_equal('engine-degraded-precondition', base.bool_call(base.ENGINE, 'degradedMode()'), False)
        set_external_adapters()
        accounts = [account.lower() for account in base.rpc('eth_accounts', [])]
        if not accounts:
            raise AssertionError('Anvil returned no unlocked account')
        sender = accounts[0]
        lp_actor = base.deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-lp-actor')
        long_actor = base.deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-long-actor')
        short_actor = base.deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-short-actor')
        guard = base.deploy(sender, 'RenounceGuardControl.sol', 'RenounceGuardControl', 'deploy-guard-control')
        base.record('harness-addresses', 'PASS', sender=sender, lp_actor=lp_actor, long_actor=long_actor, short_actor=short_actor, guard=guard)
        base.actor_send(sender, lp_actor, 'fundAndRequestJuniorDeposit(uint256)', (base.LP_ASSETS,), 'lp-request-deposit')
        deposit_id = base.uint_call(lp_actor, 'lastDepositId()')
        deposit_start = base.uint_call(base.VAULT, 'depositEpochStart(uint256)', deposit_id)
        base.advance_to(deposit_start, 'lp-deposit-maturity')
        base.actor_send(sender, lp_actor, 'settleLpEpoch(uint256)', (base.PRICE_ENTRY,), 'lp-deposit-settlement')
        claimable_assets = base.uint_call(base.VAULT, 'claimableDepositRequest(uint256,address)', deposit_id, lp_actor)
        base.assert_equal('lp-deposit-claimable-assets', claimable_assets, base.LP_ASSETS)
        base.actor_send(sender, lp_actor, 'claimJuniorDepositAll()', (), 'lp-claim-deposit')
        base.assert_positive('lp-victim-shares', base.uint_call(lp_actor, 'juniorShares()'))
        base.advance(3601, 'lp-deposit-cooldown')
        base.actor_send(sender, long_actor, 'fundTrader(uint256)', (base.LONG_FUNDS,), 'fund-long-trader')
        base.actor_send(sender, short_actor, 'fundTrader(uint256)', (base.SHORT_FUNDS,), 'fund-short-trader')
        open_positions(sender, long_actor, short_actor)
        snapshot = base.rpc('evm_snapshot', [])
        base.record('pre-renunciation-snapshot', 'PASS', snapshot=snapshot)
        base.impersonated_owner_send(base.calldata('renounceOwnership()'), 'renounced-owner-branch-renounce')
        base.assert_equal('renounced-owner-is-zero', base.address_call(base.ENGINE, 'owner()'), '0x' + '0' * 40)
        close_positions(sender, long_actor, short_actor, 'renounced-owner-branch')
        renounced_redeem = base.request_mature_redeem(sender, lp_actor, 'renounced-owner-branch')
        base.probe_blocked_settlement(sender, lp_actor, renounced_redeem, 'renounced-owner-branch')
        base.actor_send(sender, lp_actor, 'probeClearDegradedMode()', (), 'renounced-owner-branch-probe-clear')
        base.assert_equal('renounced-owner-branch-clear-call-failed', base.bool_call(lp_actor, 'lastProbeOk()'), False)
        base.assert_equal('renounced-owner-branch-clear-selector', base.bytes4_call(lp_actor, 'lastProbeSelector()'), base.selector('OwnableUnauthorizedAccount(address)'))
        base.assert_equal('snapshot-revert', base.rpc('evm_revert', [snapshot]), True)
        base.assert_equal('owner-intact-restored-owner', base.address_call(base.ENGINE, 'owner()'), base.OWNER)
        base.assert_equal('owner-intact-restored-degraded', base.bool_call(base.ENGINE, 'degradedMode()'), False)
        close_positions(sender, long_actor, short_actor, 'owner-intact-branch')
        intact_redeem = base.request_mature_redeem(sender, lp_actor, 'owner-intact-branch')
        base.probe_blocked_settlement(sender, lp_actor, intact_redeem, 'owner-intact-branch')
        base.impersonated_owner_send(base.calldata('clearDegradedMode()'), 'owner-intact-branch-clear')
        base.assert_equal('owner-intact-branch-degraded-cleared', base.bool_call(base.ENGINE, 'degradedMode()'), False)
        base.actor_send(sender, lp_actor, 'settleLpEpoch(uint256)', (base.PRICE_ENTRY,), 'owner-intact-branch-settle')
        claimable_shares = base.uint_call(base.VAULT, 'claimableRedeemRequest(uint256,address)', intact_redeem, lp_actor)
        base.assert_positive('owner-intact-branch-claimable-shares', claimable_shares)
        balance_before = base.uint_call(lp_actor, 'tokenBalance()')
        base.actor_send(sender, lp_actor, 'claimJuniorRedeemAll()', (), 'owner-intact-branch-claim')
        balance_after = base.uint_call(lp_actor, 'tokenBalance()')
        base.assert_positive('owner-intact-branch-lp-payment', balance_after - balance_before)
        send(sender, guard, base.calldata('renounceOwnership()'), 'renunciation-guard-control', expected_status=0)
        base.assert_equal('renunciation-guard-owner-retained', base.address_call(guard, 'owner()'), sender)
        result = {'result': 'PASS', 'chain_id': base.CHAIN_ID, 'fork_block': base.FORK_BLOCK, 'transactions': base._tx_count, 'lp_actor': lp_actor, 'long_actor': long_actor, 'short_actor': short_actor, 'guard_control': guard}
        base.record('run-result', 'PASS', **result)
        with open(base.RESULT, 'x', encoding='utf-8') as handle:
            json.dump(result, handle, indent=2, sort_keys=True)
            handle.write('\n')
        return 0
    except Exception as error:
        result['error_type'] = type(error).__name__
        result['error'] = str(error)
        base.record('run-result', 'FAIL', **result)
        raise
    finally:
        base.stop_process('anvil', anvil)
        base.stop_process('proxy', proxy)
        if anvil_stdout is not None:
            anvil_stdout.close()
        if proxy_stdout is not None:
            proxy_stdout.close()
        if base._run_handle is not None:
            base._run_handle.close()
if __name__ == '__main__':
    sys.exit(main())
```

### `executed/run_once_v7.py`

```python
from __future__ import annotations
import argparse
import importlib.util
import json
import pathlib
import subprocess
import sys
CODE = pathlib.Path(__file__).resolve().parent
V4_PATH = CODE / 'run_once_v4.py'
PRE_REGISTRATION = CODE.parent / 'PRE_REGISTERED_V7.md'
LOG_DIR = pathlib.Path('/data/logs/plether-af001-local-fork-2026-11-02-codex-r06')
ENGINE_LENS = '0xfca322c6868cef53bd5c344628f9874712e48108'
MINIMAL_LP_ASSETS = 1 * 10 ** 6

def _load_v4():
    spec = importlib.util.spec_from_file_location('af001_consumed_v4_runner', V4_PATH)
    if spec is None or spec.loader is None:
        raise RuntimeError('unable to load consumed V4 runner')
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module
v4 = _load_v4()
base = v4.base
base.LOG_DIR = LOG_DIR
base.RUN_LOG = LOG_DIR / 'run-final.jsonl'
base.RESULT = LOG_DIR / 'result-final.json'
base.PROXY_AUDIT = LOG_DIR / 'proxy-final.jsonl'
base.PROXY_STDOUT = LOG_DIR / 'proxy-final.stdout.log'
base.ANVIL_STDOUT = LOG_DIR / 'anvil-final.log'
base.LP_ASSETS = MINIMAL_LP_ASSETS

def signed(value: int) -> int:
    return value - (1 << 256) if value >= 1 << 255 else value

def close_preview(account: str) -> dict[str, int | bool]:
    values = base.words(base.call_raw(ENGINE_LENS, 'previewClose(address,uint256,uint256)', account, base.SIZE_400K, base.PRICE_STRESS))
    if len(values) != 26:
        raise AssertionError(f'previewClose: expected 26 words, got {len(values)}')
    return {'valid': bool(values[0]), 'invalid_reason': values[1], 'execution_price': values[2], 'size_delta': values[3], 'realized_pnl_assets': signed(values[4]), 'vpi_delta_assets': signed(values[5]), 'vpi_assets': values[6], 'execution_fee_assets': values[7], 'fresh_trader_payout_assets': values[8], 'immediate_payout_assets': values[11], 'trader_claim_balance_assets': values[12], 'remaining_size': values[14], 'remaining_margin': values[15], 'triggers_degraded_mode': bool(values[16]), 'post_op_degraded_mode': bool(values[17]), 'effective_assets_after_assets': values[18], 'max_liability_after_assets': values[19], 'payout_to_free_settlement_assets': values[24], 'deferred_payout_assets': values[25]}

def require_latching_preview(long_actor: str) -> dict[str, int | bool]:
    pool_assets_before = base.uint_call(base.POOL, 'totalAssets()')
    preview = close_preview(long_actor)
    passed = preview['valid'] and preview['invalid_reason'] == 0 and preview['triggers_degraded_mode'] and preview['post_op_degraded_mode'] and (preview['effective_assets_after_assets'] < preview['max_liability_after_assets'])
    base.record('pre-branch-long-stress-close-preview', 'PASS' if passed else 'FAIL', pool_assets_before=pool_assets_before, long_liability_before=base.side_liability(0), short_liability_before=base.side_liability(1), preview=preview)
    if not passed:
        raise AssertionError('pre-branch deployed-lens preview did not prove the frozen latch criteria')
    return preview

def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument('--ack-final-run', action='store_true')
    args = parser.parse_args()
    if not args.ack_final_run:
        parser.error('--ack-final-run is required')
    LOG_DIR.mkdir(parents=True, exist_ok=True)
    collisions = [str(path) for path in base._exclusive_paths() if path.exists()]
    if collisions:
        raise SystemExit('final evidence path already exists: ' + ', '.join(collisions))
    base._tx_count = 0
    base._run_handle = open(base.RUN_LOG, 'x', encoding='utf-8')
    proxy = None
    anvil = None
    proxy_stdout = None
    anvil_stdout = None
    result = {'result': 'FAIL', 'fork_block': base.FORK_BLOCK, 'chain_id': base.CHAIN_ID}
    try:
        base.record('run-start', 'PASS', argv=[pathlib.Path(sys.argv[0]).name, '--ack-final-run'], python=sys.version, cast_version=base.cast('--version'), anvil_version=subprocess.check_output([str(base.ANVIL), '--version'], text=True).strip(), local_tx_gas_ceiling=v4.LOCAL_TX_GAS, minimal_lp_assets=MINIMAL_LP_ASSETS, source_hashes={path.name: base.file_sha256(path) for path in sorted((CODE / 'src').glob('*.sol'))} | {'run_once.py': base.file_sha256(v4.BASE_PATH), 'run_once_v4.py': base.file_sha256(V4_PATH), 'run_once_v7.py': base.file_sha256(pathlib.Path(__file__)), 'PRE_REGISTERED_V7.md': base.file_sha256(PRE_REGISTRATION), 'release-manifest.json': base.file_sha256(base.MANIFEST)})
        proxy, anvil, proxy_stdout, anvil_stdout = base.start_processes()
        base.assert_equal('chain-id', int(base.rpc('eth_chainId', []), 16), base.CHAIN_ID)
        base.assert_equal('fork-block', int(base.rpc('eth_blockNumber', []), 16), base.FORK_BLOCK)
        base.manifest_runtime_check()
        base.assert_equal('engine-owner-precondition', base.address_call(base.ENGINE, 'owner()'), base.OWNER)
        base.assert_equal('engine-degraded-precondition', base.bool_call(base.ENGINE, 'degradedMode()'), False)
        v4.set_external_adapters()
        accounts = [account.lower() for account in base.rpc('eth_accounts', [])]
        if not accounts:
            raise AssertionError('Anvil returned no unlocked account')
        sender = accounts[0]
        lp_actor = base.deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-lp-actor')
        long_actor = base.deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-long-actor')
        short_actor = base.deploy(sender, 'ForkActor.sol', 'ForkActor', 'deploy-short-actor')
        guard = base.deploy(sender, 'RenounceGuardControl.sol', 'RenounceGuardControl', 'deploy-guard-control')
        base.record('harness-addresses', 'PASS', sender=sender, lp_actor=lp_actor, long_actor=long_actor, short_actor=short_actor, guard=guard)
        base.actor_send(sender, lp_actor, 'fundAndRequestJuniorDeposit(uint256)', (MINIMAL_LP_ASSETS,), 'lp-request-minimal-deposit')
        deposit_id = base.uint_call(lp_actor, 'lastDepositId()')
        deposit_start = base.uint_call(base.VAULT, 'depositEpochStart(uint256)', deposit_id)
        base.advance_to(deposit_start, 'lp-minimal-deposit-maturity')
        base.actor_send(sender, lp_actor, 'settleLpEpoch(uint256)', (base.PRICE_ENTRY,), 'lp-minimal-deposit-settlement')
        claimable_assets = base.uint_call(base.VAULT, 'claimableDepositRequest(uint256,address)', deposit_id, lp_actor)
        base.assert_equal('lp-minimal-deposit-claimable-assets', claimable_assets, MINIMAL_LP_ASSETS)
        base.actor_send(sender, lp_actor, 'claimJuniorDepositAll()', (), 'lp-minimal-claim-deposit')
        base.assert_positive('lp-minimal-victim-shares', base.uint_call(lp_actor, 'juniorShares()'))
        base.advance(3601, 'lp-minimal-deposit-cooldown')
        base.actor_send(sender, long_actor, 'fundTrader(uint256)', (base.LONG_FUNDS,), 'fund-long-trader')
        base.actor_send(sender, short_actor, 'fundTrader(uint256)', (base.SHORT_FUNDS,), 'fund-short-trader')
        v4.open_positions(sender, long_actor, short_actor)
        preview = require_latching_preview(long_actor)
        snapshot = base.rpc('evm_snapshot', [])
        base.record('pre-renunciation-snapshot', 'PASS', snapshot=snapshot)
        base.impersonated_owner_send(base.calldata('renounceOwnership()'), 'renounced-owner-branch-renounce')
        base.assert_equal('renounced-owner-is-zero', base.address_call(base.ENGINE, 'owner()'), '0x' + '0' * 40)
        v4.close_positions(sender, long_actor, short_actor, 'renounced-owner-branch')
        renounced_redeem = base.request_mature_redeem(sender, lp_actor, 'renounced-owner-branch')
        base.probe_blocked_settlement(sender, lp_actor, renounced_redeem, 'renounced-owner-branch')
        base.actor_send(sender, lp_actor, 'probeClearDegradedMode()', (), 'renounced-owner-branch-probe-clear')
        base.assert_equal('renounced-owner-branch-clear-call-failed', base.bool_call(lp_actor, 'lastProbeOk()'), False)
        base.assert_equal('renounced-owner-branch-clear-selector', base.bytes4_call(lp_actor, 'lastProbeSelector()'), base.selector('OwnableUnauthorizedAccount(address)'))
        base.assert_equal('snapshot-revert', base.rpc('evm_revert', [snapshot]), True)
        base.assert_equal('owner-intact-restored-owner', base.address_call(base.ENGINE, 'owner()'), base.OWNER)
        base.assert_equal('owner-intact-restored-degraded', base.bool_call(base.ENGINE, 'degradedMode()'), False)
        v4.close_positions(sender, long_actor, short_actor, 'owner-intact-branch')
        intact_redeem = base.request_mature_redeem(sender, lp_actor, 'owner-intact-branch')
        base.probe_blocked_settlement(sender, lp_actor, intact_redeem, 'owner-intact-branch')
        base.impersonated_owner_send(base.calldata('clearDegradedMode()'), 'owner-intact-branch-clear')
        base.assert_equal('owner-intact-branch-degraded-cleared', base.bool_call(base.ENGINE, 'degradedMode()'), False)
        base.actor_send(sender, lp_actor, 'settleLpEpoch(uint256)', (base.PRICE_ENTRY,), 'owner-intact-branch-settle')
        claimable_shares = base.uint_call(base.VAULT, 'claimableRedeemRequest(uint256,address)', intact_redeem, lp_actor)
        base.assert_positive('owner-intact-branch-claimable-shares', claimable_shares)
        balance_before = base.uint_call(lp_actor, 'tokenBalance()')
        base.actor_send(sender, lp_actor, 'claimJuniorRedeemAll()', (), 'owner-intact-branch-claim')
        balance_after = base.uint_call(lp_actor, 'tokenBalance()')
        base.assert_positive('owner-intact-branch-lp-payment', balance_after - balance_before)
        v4.send(sender, guard, base.calldata('renounceOwnership()'), 'renunciation-guard-control', expected_status=0)
        base.assert_equal('renunciation-guard-owner-retained', base.address_call(guard, 'owner()'), sender)
        result = {'result': 'PASS', 'chain_id': base.CHAIN_ID, 'fork_block': base.FORK_BLOCK, 'transactions': base._tx_count, 'minimal_lp_assets': MINIMAL_LP_ASSETS, 'preview': preview, 'lp_actor': lp_actor, 'long_actor': long_actor, 'short_actor': short_actor, 'guard_control': guard}
        base.record('run-result', 'PASS', **result)
        with open(base.RESULT, 'x', encoding='utf-8') as handle:
            json.dump(result, handle, indent=2, sort_keys=True)
            handle.write('\n')
        return 0
    except Exception as error:
        result['error_type'] = type(error).__name__
        result['error'] = str(error)
        base.record('run-result', 'FAIL', **result)
        raise
    finally:
        base.stop_process('anvil', anvil)
        base.stop_process('proxy', proxy)
        if anvil_stdout is not None:
            anvil_stdout.close()
        if proxy_stdout is not None:
            proxy_stdout.close()
        if base._run_handle is not None:
            base._run_handle.close()
if __name__ == '__main__':
    sys.exit(main())
```

### `executed/src/ForkActor.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IAsyncTrancheVault} from "@plether/perps/interfaces/IAsyncTrancheVault.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

interface IMockSettlementToken {

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

interface IMarginDepositor {

    function deposit(
        address account,
        uint256 amount
    ) external;

}

interface IShareBalance {

    function balanceOf(
        address account
    ) external view returns (uint256);

}

interface IReleaseRouter {

    function commitOrder(
        OrderV3Types.OrderRequest calldata request
    ) external returns (uint64 orderId);

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable;

    function settleLpEpoch(
        bytes[] calldata pythUpdateData
    ) external payable;

    function maxExecutionWindowSeconds() external view returns (uint256);

    function lifecycleBook() external view returns (IOrderLifecycleBook);

}

/// @notice Ordinary contract account used to exercise deployed public integration surfaces on local Anvil.
contract ForkActor {

    address internal constant TOKEN = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;

    uint256 public nonce;
    uint64 public lastOrderId;
    uint256 public lastDepositId;
    uint256 public lastRedeemId;
    bool public lastProbeOk;
    bytes4 public lastProbeSelector;
    bytes32 public lastProbeDataHash;

    event Probe(bytes32 indexed label, bool ok, bytes4 selector, bytes32 dataHash);

    function fundTrader(
        uint256 amount
    ) external {
        IMockSettlementToken(TOKEN).mint(address(this), amount);
        require(IMockSettlementToken(TOKEN).approve(CLEARINGHOUSE, amount), "clearinghouse approval");
        IMarginDepositor(CLEARINGHOUSE).deposit(address(this), amount);
    }

    function fundAndRequestJuniorDeposit(
        uint256 amount
    ) external returns (uint256 requestId) {
        IMockSettlementToken(TOKEN).mint(address(this), amount);
        require(IMockSettlementToken(TOKEN).approve(JUNIOR_VAULT, amount), "vault approval");
        requestId = IAsyncTrancheVault(JUNIOR_VAULT).requestDeposit(amount, address(this), address(this));
        lastDepositId = requestId;
    }

    function claimJuniorDepositAll() external returns (uint256 shares) {
        uint256 assets = IAsyncTrancheVault(JUNIOR_VAULT).claimableDepositRequest(lastDepositId, address(this));
        require(assets != 0, "no claimable deposit");
        shares = IAsyncTrancheVault(JUNIOR_VAULT).claimDeposit(lastDepositId, assets, address(this), address(this));
    }

    function requestAllJuniorRedeem() external returns (uint256 requestId) {
        uint256 shares = IAsyncTrancheVault(JUNIOR_VAULT).maxRequestRedeem(address(this));
        require(shares != 0, "no redeemable shares");
        requestId = IAsyncTrancheVault(JUNIOR_VAULT).requestRedeem(shares, address(this), address(this));
        lastRedeemId = requestId;
    }

    function claimJuniorRedeemAll() external returns (uint256 assets) {
        uint256 shares = IAsyncTrancheVault(JUNIOR_VAULT).claimableRedeemRequest(lastRedeemId, address(this));
        require(shares != 0, "no claimable redemption");
        assets = IAsyncTrancheVault(JUNIOR_VAULT).claimRedeem(lastRedeemId, shares, address(this), address(this));
    }

    function commitOpen(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) external returns (uint64 orderId) {
        return _commit(side, false, size, margin);
    }

    function commitClose(
        CfdTypes.Side side,
        uint256 size
    ) external returns (uint64 orderId) {
        return _commit(side, true, size, 0);
    }

    function executeLast(
        uint256 desiredBasketPrice
    ) external {
        bytes[] memory data = _priceData(desiredBasketPrice);
        IReleaseRouter(ROUTER).executeOrder(lastOrderId, data);
    }

    function settleLpEpoch(
        uint256 desiredBasketPrice
    ) external {
        bytes[] memory data = _priceData(desiredBasketPrice);
        IReleaseRouter(ROUTER).settleLpEpoch(data);
    }

    function probeSettleLpEpoch(
        uint256 desiredBasketPrice
    ) external returns (bool ok, bytes4 selector) {
        bytes[] memory data = _priceData(desiredBasketPrice);
        bytes memory result;
        (ok, result) = ROUTER.call(abi.encodeCall(IReleaseRouter.settleLpEpoch, (data)));
        selector = _selector(result);
        _recordProbe(keccak256("settle-lp-epoch"), ok, selector, result);
    }

    function probeClearDegradedMode() external returns (bool ok, bytes4 selector) {
        bytes memory result;
        (ok, result) = ENGINE.call(abi.encodeWithSignature("clearDegradedMode()"));
        selector = _selector(result);
        _recordProbe(keccak256("clear-degraded-mode"), ok, selector, result);
    }

    function tokenBalance() external view returns (uint256) {
        return IMockSettlementToken(TOKEN).balanceOf(address(this));
    }

    function juniorShares() external view returns (uint256) {
        return IShareBalance(JUNIOR_VAULT).balanceOf(address(this));
    }

    function _commit(
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin
    ) private returns (uint64 orderId) {
        uint256 window = IReleaseRouter(ROUTER).maxExecutionWindowSeconds();
        require(window <= type(uint32).max, "window overflow");
        require(block.timestamp <= type(uint64).max - window, "submit time overflow");
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode(address(this), ++nonce, side, close, size, margin));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = close;
        request.targetPrice = (side == CfdTypes.Side.LONG) == close ? 2e8 : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            // Both casts are proven by the two bounds immediately above.
            // forge-lint: disable-next-line(unsafe-typecast)
            submitBy: uint64(block.timestamp + window),
            // forge-lint: disable-next-line(unsafe-typecast)
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: IReleaseRouter(ROUTER).lifecycleBook().currentExecutionConfigHash(),
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
        orderId = IReleaseRouter(ROUTER).commitOrder(request);
        lastOrderId = orderId;
    }

    function _priceData(
        uint256 desiredBasketPrice
    ) private pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(desiredBasketPrice);
    }

    function _recordProbe(
        bytes32 label,
        bool ok,
        bytes4 selector,
        bytes memory result
    ) private {
        lastProbeOk = ok;
        lastProbeSelector = selector;
        lastProbeDataHash = keccak256(result);
        emit Probe(label, ok, selector, lastProbeDataHash);
    }

    function _selector(
        bytes memory result
    ) private pure returns (bytes4 selector) {
        if (result.length >= 4) {
            assembly ("memory-safe") {
                selector := mload(add(result, 0x20))
            }
        }
    }

}
```

### `executed/src/LocalArbSysAdapter.sol`

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

### `executed/src/LocalPythAdapter.sol`

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

    function getPriceUnsafe(
        bytes32 id
    ) external view returns (PythStructs.Price memory price) {
        _requireKnownId(id);
        return PythStructs.Price({price: 100_000_000, conf: 0, expo: -8, publishTime: block.timestamp});
    }

    function getPriceNoOlderThan(
        bytes32 id,
        uint256
    ) external view returns (PythStructs.Price memory price) {
        _requireKnownId(id);
        return PythStructs.Price({price: 100_000_000, conf: 0, expo: -8, publishTime: block.timestamp});
    }

    function updatePriceFeeds(
        bytes[] calldata updateData
    ) external payable {
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

    function getUpdateFee(
        bytes[] calldata
    ) external pure returns (uint256 feeAmount) {
        return 0;
    }

    function _feeds(
        bytes[] calldata updateData,
        bytes32[] calldata priceIds,
        uint64 publishTime
    ) private pure returns (PythStructs.PriceFeed[] memory priceFeeds) {
        require(updateData.length == 1, "one local price blob required");
        require(priceIds.length == 6, "unexpected feed count");
        uint256 desiredBasketPrice = abi.decode(updateData[0], (uint256));
        require(desiredBasketPrice == 100_000_000 || desiredBasketPrice == 20_000_000, "unregistered local price");

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

    function _rawPrice(
        uint256 index,
        uint256 desiredBasketPrice
    ) private pure returns (int64) {
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

    function _requireKnownId(
        bytes32 id
    ) private pure {
        for (uint256 i; i < 6; ++i) {
            if (id == _feedId(i)) {
                return;
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

    function _basePrice(
        uint256 index
    ) private pure returns (uint256) {
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

    function _inverted(
        uint256 index
    ) private pure returns (bool) {
        require(index < 6, "inversion index");
        return index != 0 && index != 2;
    }

}
```

### `executed/src/RenounceGuardControl.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/// @notice Supplementary behavior control for the one-function remediation used by other Plether governance owners.
contract RenounceGuardControl is Ownable {

    error OwnershipRenunciationDisabled();

    constructor() Ownable(msg.sender) {}

    function renounceOwnership() public view override onlyOwner {
        revert OwnershipRenunciationDisabled();
    }

}
```


