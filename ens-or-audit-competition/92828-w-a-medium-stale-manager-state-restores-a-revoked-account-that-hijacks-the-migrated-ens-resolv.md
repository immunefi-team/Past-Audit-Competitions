# 92828 w a medium stale manager state restores a revoked account that hijacks the migrated ens resolver

**Submitted on Sep 12th 2026 at 16:37:46 UTC by @hetroublemakr for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92828
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

The Manager derives an unwrapped name's legacy manager from the V1 subgraph and caches that response for five minutes. If the registrant then revokes a compromised manager using the ordinary `BaseRegistrar.reclaim(tokenId, registrant)` recovery path, the migration flow never reads the current V1 `ENSRegistry.owner(node)` before execution. Its ownership preflight checks only the BaseRegistrar NFT owner, and its final pre-wallet freshness assertion checks only which temporary approvals are missing.

The stale address therefore remains in `managerAddress`. The Manager constructs one atomic HCA batch that migrates the name and calls:

```solidity
ETHRegistryV2.grantRoles(labelhash, 1 << 24, revokedManager);
```

The bit is `ROLE_SET_RESOLVER`. Post-state verification treats the stale grant as required success, while cleanup revokes only the HCA's temporary operator approval. The revoked manager consequently regains persistent resolver authority in V2 and can point the migrated name at an attacker-controlled resolver.

The fixed-fork PoC proves the full transition using current Sepolia contracts and the live `tkn.eth` state. The victim first reclaims the V1 manager role from the compromised account. The exact app-equivalent HCA batch then migrates the name and restores the stale account. After Manager-style cleanup, the former manager installs a malicious resolver whose ETH address record returns the former manager. The paired control migrates the same name without the stale grant and proves the former manager's resolver change reverts.

### Threat model and required interaction

The attacker is a formerly authorized V1 manager that the registrant has already revoked. At exploit time, the attacker needs no NFT ownership, approval, HCA authority, wallet key, session, or V2 role. The victim must have loaded the Manager while that address was still indexed as manager, reclaim control from the compromised manager, and complete the normal migration while the cached snapshot remains in use. The app's own five-minute fresh-cache interval makes this deterministic without requiring a subgraph outage; ordinary index lag can extend the race.

The victim performs the expected migration wallet confirmations. This report does not claim a zero-interaction attack. It claims the program's explicit Critical migration-hijack impact: a normal Manager migration silently assigns resolver authority to an account the registrant just revoked.

## Impact

A compromised V1 manager that the registrant has successfully removed can regain persistent V2 `ROLE_SET_RESOLVER` authority solely because the victim uses the normal Manager migration flow before its cached authority snapshot is refreshed. The revoked account can then replace the resolver with attacker-controlled code, causing the migrated ENS name's ETH address record to resolve to the attacker. It can alternatively point to a non-resolver and brick resolution.

This is not a claim of NFT ownership theft or a demonstrated public-chain fund transfer. It is the program's exact Critical impact 6286: the Manager's v1-to-v2 migration assigns the name's resolver authority to an attacker and enables name hijacking. The stale role remains after the app's advertised temporary-access cleanup and requires a separate V2 revocation by the victim.

## Link to Proof of Concept

<https://gist.github.com/hetroublehacker/6368619d5ee167ab9b96c0210144ae29>

## Proof of Concept

1. Save the single gist file as `FindingStandalone.t.sol` and optionally verify its SHA-256:

```bash
sha256sum FindingStandalone.t.sol
```

Expected digest:

```
9d00abda1cb84fe92ca67289b4c4d7e61c90a190664d13f1a20be527cd7becc1
```

2. Put the file in an otherwise empty Foundry test directory. It has no imports, so no repository clone, `forge-std`, Node.js, pnpm, or ENS dependencies are needed:

```bash
mkdir -p finding/test
cp FindingStandalone.t.sol finding/test/
```

3. With Foundry and Internet access to an archive-capable Sepolia RPC, run the two tests:

```bash
forge test --root finding --match-contract FindingStandaloneTest -vv
```

The file defaults to `https://ethereum-sepolia-rpc.publicnode.com`. To use a different endpoint:

```bash
SEPOLIA_RPC_URL='https://YOUR_SEPOLIA_ARCHIVE_RPC' \
  forge test --root finding --match-contract FindingStandaloneTest -vv
```

4. Confirm the exploit and negative control both pass:

```
Ran 2 tests for test/FindingStandalone.t.sol:FindingStandaloneTest
[PASS] testControl_NoStaleGrantMeansFormerManagerCannotSetResolver()
[PASS] testExploit_StalePlanRestoresRevokedManagerAndHijacksResolver()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

5. Inspect the logs. They show cached manager `0x741d...d08`, authoritative manager `0x3cDB...c6De` after victim-authorized `reclaim()`, the stale V2 role surviving HCA cleanup, and the attacker-controlled ETH resolution. The control reports a zero stale-manager role and rejects the same resolver change.

All writes remain confined to the disposable fixed-block fork. The public endpoint is used only for historical reads; no transaction is signed or broadcast.
