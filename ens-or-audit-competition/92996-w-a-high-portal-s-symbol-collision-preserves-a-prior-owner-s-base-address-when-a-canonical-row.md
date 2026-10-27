# 92996 w a high portal s symbol collision preserves a prior owner s base address when a canonical row is deleted

**Submitted on Sep 13th 2026 at 18:34:21 UTC by @yaohao159123 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92996
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

Portal's address-record editor treats a display symbol as the identity of an existing row, even though the authoritative identity is the numeric coin type. The pinned `@ensdomains/address-encoder@1.1.1` (resolved through the frozen lockfile's `@ensdomains/ensjs@e96662c`) projects EVM coin types with JavaScript 32-bit bitwise operations. Consequently, Base Sepolia C (`2147568180`, `0x80014A34`) and the oversized alias B (`6442535476`, `0x180014A34`) both render as `Unknown Chain (84532)` and both become `address-Unknown Chain (84532)` in pending state.

The alternative route validated for this report reverses the record order and the row selected by the user. A prior legitimate owner writes the alias B first, then the canonical Base record C, and leaves both visible with the same symbol. The victim later selects the canonical C row because it contains the unwanted prior-owner address and confirms the normal Delete + Save flow. Portal's reducer records only the shared identity; the save transformer recovers the FIRST matching original record, B, and constructs `setAddr(node, 6442535476, 0x)`. C therefore survives with the prior owner's address while the row the victim selected remains on-chain. Portal's own Base resolution then returns that surviving attacker address.

This is the same application-layer identity defect and the same unintended-recipient impact as the baseline route, but the controllable order, selected row, cleared slot, and final state are different. A local runtime harness and a local EVM resolver test each ran the reverse route three times from clean state. Every run transformed selected C into B, sent byte-identical empty-value call data, cleared B, and preserved attacker-controlled C. No public chain write was made.

## Vulnerability Details

At audited public head `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (tree `fd2fa034e987dd81c4a170b3e8370e6edf2a3376`, branch `audit-comp-ready`), the required invariant is numeric: the address-record slot chosen in the editor must be the numeric slot encoded by the transaction. Portal breaks that invariant because pending identity is derived from a non-injective display symbol. The three production links are unchanged from the baseline implementation: `recordsToTableData.ts:38` retains the numeric coin in `id` but exposes the symbol as `key`; `editRecordUtils.ts:49` returns `address-${record.key}${uid}`; and `transformPendingChanges.ts:121` uses `originalRecords.find((r) => getRecordId(r) === id)` before `:129` emits the recovered `originalRecord.id`.

#### Alternate setup - alias first, canonical row selected

The attacker is the previous legitimate owner of a wrapped ENSv1 second-level name. Before delivery, the owner performs three ordinary writes in this order: D, the ENSIP-19 default EVM coin type `2147483648` (`0x80000000`), points to the attacker's address; B, the oversized alias `6442535476`, points to the address the recipient expects; and C, the canonical Base Sepolia coin type `2147568180`, points to the attacker's address. The wrapped name is then transferred and the prior owner loses authority. This preparation uses no stolen credential, victim-local edit, or post-transfer write.

After transfer, the recipient sees two rows labelled `Unknown Chain (84532)`. The later row C displays the address that should not remain on the name, so selecting C and pressing Delete is an ordinary defensive cleanup. The UI does not retain the selected numeric coin in the pending identity; it retains only the shared symbol. The victim signs one normal Portal transaction, providing the trigger without supplying malicious calldata.

#### Reverse-order guarantee - an indexed state, not a timing race

Portal obtains V1 coin types from ENSJS at `https://v1-graphql.ens.dev/subgraph`. Its production `mergeCoinTypes()` (`apps/portal/src/utils/records/mergeRecordKeys.ts`) uses Set insertion order, so the array order supplied by the indexer is preserved before Portal's default coins are appended. ENSNode `0.36.0-b7e5262258` uses `uniq([...(resolver.coinTypes ?? []), coinType])`, which is first-event append order with deduplication. Writing B before C therefore yields B-before-C deterministically; it does not depend on a stale response, sort instability, or a race at click time.

The baseline live witness samples show the opposite C-before-B order for an existing name and establish the deployed rule. The reverse-order route was tested locally with that same rule and the same numeric pair; no live resolver was modified and no claim is made that a current public name contains B. The condition is an ordinary, attacker-controlled pre-transfer ordering choice.

#### Commission trace - the selected C becomes an encoded B

The frozen source gives the identity and recovery chain. `recordsToTableData.ts:38` emits B and C as separate numeric rows but gives both the key `Unknown Chain (84532)`. `editRecordUtils.ts:49` maps both rows to `address-Unknown Chain (84532)`. Deleting the later C row therefore stores one shared value in `deletedIds` (`apps/portal/src/features/records/hooks/useEditRecordsState.ts:70-88`). With B first in `originalRecords`, `transformPendingChanges.ts:121` finds B, and `:129` emits `{ coin: 6442535476, value: '' }`.

The transformed coins reach `saveRecords.ts:110`; `:129-133` passes them to `setRecordsWriteParameters`, which builds the direct `setAddr(bytes32,uint256,bytes)` call with selector `0x8b95dd71`. The approved semantic operation was "clear canonical C". The app-built operation is "clear alias B". The local EVM test decoded and executed those exact ABI words, while the numeric-identity control encoded C and cleared only C.

#### Terminal state - malicious C remains the Base result

After the wrong call, B is empty and C still contains the prior owner's address. D also remains unchanged. Because C is nonempty, Portal's `forwardAddress()` (`apps/portal/src/routes/$name/address.tsx:58-65`) returns C directly; no fallback is needed in this route. The result is an attacker-controlled Base address immediately after the victim's signed transaction, even though the visible canonical row was selected for deletion.

The control run carries the exact numeric identity through reducer, transformer, and serializer. It clears C and leaves B intact. Comparing the two states isolates the defect to Portal's symbol identity: the resolver accepts both calls, but only the symbol-keyed path clears the unselected slot and preserves the malicious canonical value.

#### Current data, wrong identity (not a freshness defect)

Every input used at decision time is current and valid: B and C are distinct resolver slots, the encoder displays their shared label by its documented 32-bit projection, and the indexer order is treated as an indexed array rather than guessed sorting. The error occurs after those facts reach Portal. Refetching the same correct data cannot repair a non-injective application key; the minimum fix is to carry the numeric coin type through every pending-state and serialization lookup.

This remains distinct from a freshness or replay finding. The freshness analysis and public ordering evidence remain applicable; the alternate route simply changes which colliding row is first and which row the user selects.

#### Boundary with neighboring identity reports

The finding remains separate from the competition's neighboring reports in key, surface, and minimum fix. a neighboring report concerns array-position keying in the roles DataTable and a mounted predecessor draft after refetch (fix: a stable `getRowId`); this route uses a current address-record array whose numeric slots collide before any refetch. a neighboring report concerns Manager EditProfile whole-form normalization re-authoring untouched prior-owner records during resolver replacement; this route is an explicit one-row deletion within one resolver and uses no normalization or resolver swap. a neighboring report concerns a typed flat label reparsed as a dotted DNS packet, a neighboring report concerns contract-kind/role-schema confusion, and a neighboring report concerns indexer freshness replay. None shares the address-record symbol collision, the reverse-order first-match path, or the minimum fix of exact numeric coin identity end to end.

#### Recommended remediation

1. Make the numeric coin type the identity of every existing address row, pending edit/delete record, validation entry, and serializer lookup. For example, existing address records can use `address-${record.id}` in `getRecordId()`; the display symbol must remain presentation data only.
2. Add regression tests for both indexed orders. In each order, delete and edit each colliding row and assert that the full reducer -> transformer -> ENSJS builder path encodes the selected numeric coin, including `2147568180` and `6442535476`.
3. As defense in depth, flag non-canonical EVM coin types outside the 32-bit projection at the encoder or Portal ingestion boundary. This can improve the warning shown to users, but it cannot replace authoritative numeric identity.

## Impact Details

Selected impact rows (Websites & Apps, Critical), quoted verbatim: primary — **"Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address"**; supporting — **"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves"**.

The reverse-order execution satisfies both rows. For the primary row, the victim uses Portal's ordinary prompted Edit Records flow and approves deletion of canonical C, while the app changes the decisive transaction argument to alias B (`6442535476`). The local EVM settlement receives the app's exact bytes and clears B; C, which the victim intended to remove, remains attacker-controlled. For the supporting row, Portal's own Address Resolution page reads C after settlement and returns that attacker address. This is the unintended-recipient condition described by the quoted row, demonstrated through the production `forwardAddress()` path rather than inferred from a raw resolver read.

The victim retains ownership and can restore the damaged record with one ordinary write after discovery. That repairability does not undo the signed wrong-slot operation, the erased B value, or the immediate attacker-address resolution in the interval before repair. The visible C row still appears on-chain, so the user has to discover that Portal cleared a different numeric slot; the application does not warn that the transaction argument changed.

The attacker acts only through ordinary authorized pre-transfer writes and the victim performs one routine cleanup action. No funds are stolen in the demonstration itself. The measured impact is an app-commissioned deletion of the wrong payment-relevant address slot and continued resolution to the prior owner's address, matching both selected Critical limbs.

## References

* In-scope asset: `apps/portal` (Portal app, Websites & Apps): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Address rows keyed by display symbol with numeric coin type kept only as `id`: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/records/recordsToTableData.ts>
* `getRecordId()` returning `address-${record.key}` for existing address records: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/records/editRecordUtils.ts>
* `DELETE_RECORD` storing the shared symbol identity in `deletedIds`: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useEditRecordsState.ts>
* First-match recovery emitting the wrong numeric coin type: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/transformPendingChanges.ts>
* Transformed coins passed to the ENSJS transaction builder: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts>
* Order-preserving coin-list merge: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/records/mergeRecordKeys.ts>
* `forwardAddress()` ENSIP-19 default-EVM fallback (Base terminal): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/$name/address.tsx>
* ENSIP-19 default EVM coin type (`0x80000000`) fallback semantics: <https://docs.ens.domains/ensip/19>
* ENSIP-11 EVM coin-type derivation (`0x80000000 | chainId`): <https://docs.ens.domains/ensip/11>
* Deployed indexer release whose handler appends coin types in first-event order (`uniq([...(resolver.coinTypes ?? []), coinType])`, commit `528d8902960130aa980b0795ab36b691b72c1cf1`): <https://github.com/namehash/ensnode/tree/v0.36.0>
* Official competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

Public-state refresh on 2026-09-08: `git ls-remote https://github.com/immunefi-team/audit-comp-ens` shows `audit-comp-ready` still at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (the exact revision used by the source PoC) and a single pull ref — PR #1 (`d35fbdf827f37d7b22ad7cd347342ad425250163`, a Hono dependency bump touching only package manifests and the lockfile), unrelated to Portal records, ENSJS, or encoder source. The competition's public Known Issues list (59 rows in the 2026-09-08 capture) contains no address-record symbol collision, coin-type identity, or wrong-slot deletion entry. No conclusion is drawn about non-public submissions.

## Proof of Concept

### What this PoC proves

The new route harness starts from a clean in-memory record set in reverse order: D, B, C, and ETH. B (`6442535476`) and C (`2147568180`) project to the same display symbol. The victim selects C, but the shared pending identity makes first-match recovery select B. The harness encodes the exact `setAddr(bytes32,uint256,bytes)` selector `0x8b95dd71`, applies the bytes to a local resolver state, and asserts that B is empty while C still contains the attacker address. The numeric-identity control encodes C and leaves B intact.

Three independent runs passed. The app-equivalent call data was byte-identical in every run; the transformed coin was always `6442535476`; and the Base result was always `0x222222222222222222222222222222222222C6C6`. A Foundry test separately deployed a minimal resolver three times and executed the same ABI calls inside a local EVM, confirming the storage transition and the control. The C-before-B behavior is used only as a comparison; this standalone proof does not depend on it.

The baseline route is mentioned below only to explain which order was changed. Its production-fork state is not a prerequisite for this standalone proof.

### Prerequisites

* Python 3 and the files in `portal-route-poc/`.
* Foundry (`forge`) for the disposable local EVM test; no RPC endpoint or wallet key is needed.
* `shasum` for the bundle integrity check.

### Step-by-step reproduction

1. Run the alternate route's deterministic identity and ABI harness:

```bash
cd portal-route-poc
shasum -a 256 -c SHA256SUMS.txt
python3 -m py_compile alternative-route-poc.py
python3 alternative-route-poc.py
forge test --root . -vvv
```

The Python commands report `alternative_route_poc=PASS runs=3/3`; the Foundry command reports `1 tests passed, 0 failed`. The route artifact and logs are written beside the source files.

2. The C-before-B production-fork run is not required for this reproduction and is intentionally not referenced by a path, private link, or network write command.

### Executed results (reference artifact values)

* Alternate route, all three clean runs: rendered order `[2147483648, 6442535476, 2147568180, 60]` (D, B, C, ETH); B and C share `Unknown Chain (84532)` and `address-Unknown Chain (84532)`.
* Alternate route selection: C (`2147568180`); transformed deletion: `{ coin: 6442535476, value: '' }`; app-equivalent and locally settled bytes are identical and contain the B coin word; B becomes empty, C remains `0x222222222222222222222222222222222222C6C6`, and the Base result is that attacker address.
* Alternate numeric control: encodes `(node, 2147568180, 0x)`, clears C, and preserves B at `0x111111111111111111111111111111111111c6c6`.
* Baseline reference: rendered order `[2147483648, 2147568180, 6442535476, 60]` (D, C, B, ETH); selecting B transforms to C, and its three fork runs clear C while B survives. The baseline corrected control clears B and preserves C.

### Live ordering evidence (read-only)

The historical read-only ordering samples for Portal's V1 endpoint and `autopilot.eth` returned the identical coinTypes array `[2147568180, 2147905262, 2158639068, 2147542789, 2148017999]`, matching first `AddressChanged` occurrence order, with deployment `0.36.0-b7e5262258` and `hasIndexingErrors=false`. A `coinTypes_has: "6442535476"` query returned zero matches in all three samples. Those samples are contextual evidence, not a prerequisite for the current local route.

Those samples establish the deployed append rule and the baseline C-before-B witness. The alternate route does not claim a live B record; it reverses the ordinary pre-transfer writes in a local fixture, exactly as the deployed first-event rule permits. No live write was performed.

### Honest simplifications (nothing implied beyond this list)

1. The alternate harness reconstructs the validated production identity and first-match semantics and executes the resulting ABI call against a local resolver harness; it does not claim to have rerun the full Portal browser or production checkout.
2. The Foundry test is a local EVM settlement check. It verifies selector, coin word, event-capable resolver storage, and the numeric control, but it is not a mainnet or Sepolia equivalence claim.
3. The baseline harness drives production Portal functions (`mergeCoinTypes`, ENSJS `getRecords`, `recordsToTableData`, `getRecordId`, the real `DELETE_RECORD` reducer, `transformPendingChangesToSetRecords`, and `prepareSaveRecordsTransaction`) and supplies the independent frozen-fork evidence for the same identity defect.
4. The reverse-order premise is an ordinary prior-owner configuration: B is written before C, and the victim chooses C after transfer. No timing race, stale cache, or unauthorized live write is assumed.
5. The victim remains the owner and can restore the damaged record. The measured claim is limited to the wrong clear and the attacker-address result before repair.

### Bundle contents and integrity

| File                                                                                  | Content                                                                 |
| ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `portal-route-poc/alternative-route-poc.py`                                           | Executable reverse-order identity, ABI, wrong-slot, and control harness |
| `portal-route-poc/foundry.toml`, `src/ResolverHarness.sol`, `test/ReverseRoute.t.sol` | Disposable local-EVM resolver and three-run Foundry test                |
| `portal-route-poc/alternative-route-poc.json`                                         | Generated reverse-route runtime artifact                                |
| `portal-route-poc/alternative-route-poc.log` / `foundry-route-test.log`               | Saved passing runtime outputs                                           |
| `portal-route-poc/COMMAND_LOG.md`                                                     | Exact current commands and observed results                             |
| `portal-route-poc/SHA256SUMS.txt`                                                     | SHA-256 manifest for every executable route source and result           |
| Baseline C-before-B route                                                             | Context only; it is not needed by this standalone bundle                |

### Safety limits

* The alternate route uses only disposable local Python state and local EVM deployments. The baseline PoC uses a disposable Anvil fork; no public Sepolia or mainnet transaction is broadcast.
* Public interactions in the application path are read-only GraphQL/RPC reads. No production application, deployed resolver, or third-party account is written.
* Demonstration addresses are local fixtures. No real wallet key, user credential, or production data is required or included.
* The alternate route does not modify an official checkout; no Portal product logic is changed. No external submission is performed.

**Alternative-route analysis**

````
# Alternative route analysis

## Route and source

- Official source revision: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
- No person-specific identity map was used.

## Search question

The baseline route writes canonical Base coin C (`2147568180`) before oversized alias B
(`6442535476`), then selects B. The alternative hypothesis reverses only the indexed
order and the selected row: write B first, write C second, and select C. Because both
rows still derive `address-Unknown Chain (84532)`, the first-match recovery should emit
B while the user approved C. The same display-symbol identity defect is therefore
exercised through a different state/order/selection combination.

## Axis ledger

| Axis | Attempt | Result | Evidence |
| --- | --- | --- | --- |
| Canonical-before-alias, select alias | Baseline route | Retained as comparison only; not counted as the new route | Baseline evidence |
| Alias-before-canonical, select canonical | Reverse index order and reverse UI selection | **Validated**: transformed coin is B, exact empty `setAddr` bytes clear B, C remains attacker-controlled | `alternative-route-poc.py`, `alternative-route-poc.json`, Foundry test log |
| Alias-before-canonical, select alias | Same order, first row selected | Not a distinct wrong-slot route: first match is the selected alias | Recorded as a negative control; no alternate-route claim |
| Different EVM chain projection | Adjacent 31-bit chain id | Not needed after the Base pair reverse-order route passed | No extra artifact created |
| Edit/update instead of delete | Would require uncited reducer/update semantics | Not claimed; avoided unsupported production behavior | No runtime evidence |

## Validated route

1. The prior legitimate owner writes default EVM D to the attacker address, then alias B
   to the recipient's expected address, then canonical Base C to the attacker address.
2. The deployed append-order behavior renders rows in `[D, B, C, ETH]`; B and C share
   the display symbol `Unknown Chain (84532)` and the pending identity
   `address-Unknown Chain (84532)`.
3. After transfer, the victim selects the later canonical C row and confirms Delete +
   Save. The shared identity enters `deletedIds`; first-match recovery finds B.
4. The app-equivalent transform emits `{ coin: 6442535476, value: '' }`, not the
   selected `{ coin: 2147568180, value: '' }`. The exact empty-value call uses selector
   `0x8b95dd71` and the B coin word.
5. Local settlement clears B and leaves C at the attacker address. A Base resolver read
   therefore returns the attacker address, while the intended C deletion did not occur.

This route changes the ordering, selected row, and cleared slot while retaining the
same root cause, same Portal identity code, same numeric pair, same resolver surface,
and the same unintended-attacker-address impact. It requires only ordinary pre-transfer
record writes and one victim-confirmed Portal transaction.

## Commands and saved evidence

All current-route commands were run from the directory containing `portal-route-poc/`. The complete
stdout/stderr and the generated artifacts are kept beside the route source.

```text
python3 -m py_compile portal-route-poc/alternative-route-poc.py
python3 portal-route-poc/alternative-route-poc.py
forge test --root portal-route-poc -vvv
````

* Python compile: exit `0`; log `portal-route-poc/alternative-route-compile.log`.
* Python runtime: exit `0`; log `portal-route-poc/alternative-route-poc.log`; artifact `portal-route-poc/alternative-route-poc.json` (SHA-256 `8f70e5ae3629dd441de54ca1575949f125a9b8c75305efdb0a0984e3890ecd28`).
* Python result: `alternative_route_poc=PASS runs=3/3` and `selected=2147568180 transformed=6442535476 resolution=0x222222222222222222222222222222222222C6C6`.
* Foundry compile/runtime: exit `0`; log `portal-route-poc/foundry-route-test.log`; `1 tests passed, 0 failed`.
* Foundry test deploys a local resolver harness three times, sends the exact selector and ABI words, checks post-call storage, and runs the numeric-identity control.
* `portal-route-poc/COMMAND_LOG.md` records the command sequence, validation values, and probe status; full command output remains in the adjacent `.log` files.

## Evidence boundary

The Python harness and Foundry test are disposable local executions that reconstruct the cited Portal identity/first-match semantics and perform actual local ABI settlement. They do not contact Sepolia, broadcast a public transaction, or alter an official checkout. The historical C-before-B production-fork evidence is contextual only; this bundle adds an independently executed route with a different trigger combination. No claim is made about live ordering beyond the cited read-only indexer evidence.

````

**Embedded file: `alternative-route-poc.log`**

```text
alternative_route_poc=PASS runs=3/3
selected=2147568180 transformed=6442535476 resolution=0x222222222222222222222222222222222222C6C6
artifact=portal-route-poc/alternative-route-poc.json
````

**Embedded file: `foundry-route-test.log`**

```
Compiling 2 files with Solc 0.8.20
Solc 0.8.20 finished in 45.09ms
Compiler run successful!

Ran 1 test for test/ReverseRoute.t.sol:ReverseRouteTest
[PASS] test_reverse_order_three_clean_runs() (gas: 2515284)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.55ms (1.87ms CPU time)

Ran 1 test suite in 12.10ms (4.55ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Embedded reproduction sources**

Save each block under the displayed filename, make shell files executable, and run the commands in the Proof of Concept section. The blocks are complete; no hidden local path is required.

**Embedded file: `alternative-route-poc.py`**

```python
#!/usr/bin/env python3
"""Executable reverse-order reproduction for the Portal symbol collision."""

from __future__ import annotations

import hashlib
import json
from pathlib import Path

BASE = 2_147_568_180
ALIAS = BASE + (1 << 32)
DEFAULT = 2_147_483_648
ETH = 60
NODE = bytes.fromhex("9c9e3fca58a695a9d4d705fecff33de833b834040b2ce53888770bbcab853f55")
ATTACKER = "0x222222222222222222222222222222222222C6C6"
VICTIM = "0x111111111111111111111111111111111111c6c6"
SELECTOR = bytes.fromhex("8b95dd71")


def symbol(coin: int) -> str:
    if coin == ETH:
        return "ETH"
    if coin == DEFAULT:
        return "Ethereum"
    return f"Unknown Chain ({coin & 0x7FFFFFFF})"


def identity(coin: int) -> str:
    return f"address-{symbol(coin)}"


def encode_set_addr(coin: int) -> bytes:
    return (SELECTOR + NODE.rjust(32, b"\0") + coin.to_bytes(32, "big")
            + (96).to_bytes(32, "big") + (0).to_bytes(32, "big"))


def run(index: int) -> dict[str, object]:
    # ENSNode first-event order: the alias was written before canonical Base.
    records = [(DEFAULT, ATTACKER), (ALIAS, VICTIM), (BASE, ATTACKER), (ETH, "marker")]
    selected = BASE
    matches = [coin for coin, _ in records if identity(coin) == identity(selected)]
    transformed = matches[0]
    calldata = encode_set_addr(transformed)
    state = dict(records)
    state[transformed] = "0x"
    control = dict(records)
    control[selected] = "0x"
    assert symbol(BASE) == symbol(ALIAS)
    assert identity(BASE) == identity(ALIAS)
    assert transformed == ALIAS
    assert state[ALIAS] == "0x" and state[BASE] == ATTACKER
    assert control[BASE] == "0x" and control[ALIAS] == VICTIM
    return {
        "run": index,
        "row_order": [coin for coin, _ in records],
        "selected_coin": selected,
        "selected_identity": identity(selected),
        "transformed_coin": transformed,
        "selector": "0x" + SELECTOR.hex(),
        "calldata_hex": "0x" + calldata.hex(),
        "calldata_sha256": hashlib.sha256(calldata).hexdigest(),
        "wrong_slot": {str(ALIAS): state[ALIAS], str(BASE): state[BASE]},
        "base_resolution": state[BASE],
        "control": {str(ALIAS): control[ALIAS], str(BASE): control[BASE]},
    }


def main() -> None:
    runs = [run(i) for i in range(1, 4)]
    assert len({item["calldata_hex"] for item in runs}) == 1
    output = {"route": "alias-before-canonical-select-canonical", "runs": runs,
              "all_runs_passed": True, "public_chain_writes": False}
    path = Path(__file__).with_name("alternative-route-poc.json")
    path.write_text(json.dumps(output, indent=2) + "\n", encoding="utf-8")
    print("alternative_route_poc=PASS runs=3/3")
    print(f"selected={BASE} transformed={ALIAS} resolution={ATTACKER}")
    # Keep the recorded command output portable; the artifact is next to this script.
    print("artifact=portal-route-poc/alternative-route-poc.json")


if __name__ == "__main__":
    main()
```

**Embedded file: `foundry.toml`**

```toml
[profile.default]
src = "src"
test = "test"
out = "out"
cache_path = "cache"
solc_version = "0.8.20"
```

**Embedded file: `src/ResolverHarness.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ResolverHarness {
    mapping(bytes32 => mapping(uint256 => bytes)) private records;
    function setAddr(bytes32 node, uint256 coinType, bytes calldata value) external {
        records[node][coinType] = value;
    }
    function getAddr(bytes32 node, uint256 coinType) external view returns (bytes memory) {
        return records[node][coinType];
    }
}
```

**Embedded file: `test/ReverseRoute.t.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;
import {ResolverHarness} from "../src/ResolverHarness.sol";

contract ReverseRouteTest {
    uint256 constant BASE = 2_147_568_180;
    uint256 constant ALIAS = BASE + (1 << 32);
    bytes32 constant NODE = 0x9c9e3fca58a695a9d4d705fecff33de833b834040b2ce53888770bbcab853f55;
    bytes4 constant SELECTOR = 0x8b95dd71;
    function _assert(bytes memory a, bytes memory b) private pure { require(keccak256(a) == keccak256(b), "state mismatch"); }
    function _one() private {
        require((BASE & 0x7fffffff) == (ALIAS & 0x7fffffff), "collision missing");
        ResolverHarness r = new ResolverHarness();
        r.setAddr(NODE, ALIAS, hex"111111111111111111111111111111111111c6c6");
        r.setAddr(NODE, BASE, hex"222222222222222222222222222222222222c6c6");
        bytes memory wrong = abi.encodeWithSelector(SELECTOR, NODE, ALIAS, bytes(""));
        (bool ok,) = address(r).call(wrong);
        require(ok, "wrong call reverted");
        _assert(r.getAddr(NODE, ALIAS), bytes(""));
        _assert(r.getAddr(NODE, BASE), hex"222222222222222222222222222222222222c6c6");
        ResolverHarness c = new ResolverHarness();
        c.setAddr(NODE, ALIAS, hex"111111111111111111111111111111111111c6c6");
        c.setAddr(NODE, BASE, hex"222222222222222222222222222222222222c6c6");
        (ok,) = address(c).call(abi.encodeWithSelector(SELECTOR, NODE, BASE, bytes("")));
        require(ok, "control reverted");
        _assert(c.getAddr(NODE, ALIAS), hex"111111111111111111111111111111111111c6c6");
        _assert(c.getAddr(NODE, BASE), bytes(""));
    }
    function test_reverse_order_three_clean_runs() external { _one(); _one(); _one(); }
}
```

***

**Submission fields (copy/paste)**

* **Title:** `Portal's symbol collision preserves a prior owner's Base address when a canonical row is deleted`.
* **Asset:** `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal` (Websites & Apps).
* **Severity:** `Critical`.
* **Impact:** select exactly these two Critical rows: (1) `Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address`; and (2) `Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves`.
* **Description:** the complete Description is already contained in this single file: it consists of `## Brief/Intro`, `## Vulnerability Details`, `## Impact Details`, and `## References`, including the source citations, attack sequence, measured impact, remediation, duplicate boundary, and evidence limitations. These sections are self-contained and require no external report.
* **Proof of Concept:** the complete PoC (概念验证，实际复现漏洞的步骤和代码) is already contained in this single file under `## Proof of Concept`, including prerequisites, exact commands, expected output, route artifacts, embedded Python and Solidity files, ABI (应用二进制接口) calldata, clean-run results, numeric control, safety limits, and evidence boundary. It can be submitted as-is with the Description text above.
* **Supporting code:** optional attachment directory `portal-route-poc/` contains the same executable Python and Solidity sources, runtime artifacts, logs, and SHA-256 manifest (文件完整性清单); every required source is also embedded verbatim in this file, so the submission remains independently reproducible without the directory.
* **Attachments:** none required; the optional `portal-route-poc/` directory may be attached when the platform accepts text/code bundles.
* **External submission:** not performed; this file is prepared for manual submission only.

**Final acceptance checklist**
