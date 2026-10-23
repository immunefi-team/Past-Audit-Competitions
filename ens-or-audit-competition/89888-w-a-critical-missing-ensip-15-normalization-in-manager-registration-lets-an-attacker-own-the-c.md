# 89888 w a critical missing ensip 15 normalization in manager registration lets an attacker own the canonical name while a victim buys a hidden lookalike

**Submitted on Aug 21st 2026 at 17:12:47 UTC by @jerrysd for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89888
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager does not apply ENSIP-15 normalization before availability checks or registration. It only trims and lowercases route input, then sends the raw Unicode bytes to `ETHRegistrar.register()`.

An attacker can register a canonical NFC name, then send a victim a visually identical NFD registration URL. Manager treats the NFD bytes as a separate available label. Normal ENS clients canonicalize the visible name to the attacker's NFC label, while the victim pays for an offcanonical token that Manager omits from the normal dashboard and reports as nonexistent.

## Vulnerability Details

`apps/manager/src/features/register-v2/utils/name-parser.ts:46` calls a lowercase conversion, not ENSIP-15 normalization:

```ts
const normalized = name.trim().toLowerCase()
```

The raw label survives the transaction path. `registration.hca.actors.ts:498` only removes `.eth`, and `registration-calls.ts:286-295` passes `params.label` directly to `register()`:

```ts
const cleanLabel = (name: string): string => name.replace(/\.eth$/, '')

args: [params.label, params.wallet, params.secret, ZERO_ADDRESS,
  params.resolver, params.duration, c.usdc, REFERER]
```

The registrar keys state by exact bytes: `LibLabel.id(label)` is `keccak256(bytes(label))`. Canonically equivalent strings therefore receive different hashes and independent ownership.

## Impact Details

Apply the ENSIP-15 `normalize()` implementation at the first route/input boundary and pass only that canonical result through every availability, commitment, price, namehash, resolver, and registration operation. Reject or redirect whenever the input differs from its canonical form. Also include existing unnormalized domains in a recovery view so current owners can locate and manage them.

## References

* <https://docs.ens.domains/resolution/names/>
* <https://docs.ens.domains/ensip/15/>

## Link to Proof of Concept

<https://gist.github.com/JRBusiness/620bcbfd5b8e044a3f964a1914b2b818>

## Proof of Concept

## Proof of Concept

ENSIP-15 normalization maps the first string to the second, but they look identical.

1. Open the successful public HCA registration transaction at block `11517958`:\
   <https://sepolia.etherscan.io/tx/0xeff35f330085774ab77919d5c97507dc591541af500d508387d28ee6d9c18638>

   The successful transaction registered label bytes `65cc816863612d3161303136663836313066` to `0x13D7d7267154D7712aF147F964226BC8D6B3D951` through the HCA flow.
2. Query the deployed registrar and registry:

```bash
RPC=https://ethereum-sepolia-rpc.publicnode.com
RAW=$'e\u0301hca-1a016f8610f'
NFC=$'\u00e9hca-1a016f8610f'
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
REGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2

cast keccak "$RAW"
cast keccak "$NFC"
cast call --rpc-url "$RPC" "$REGISTRAR" 'isAvailable(string)(bool)' "$RAW"
cast call --rpc-url "$RPC" "$REGISTRAR" 'isAvailable(string)(bool)' "$NFC"
cast call --rpc-url "$RPC" "$REGISTRY" 'findOwner(string)(address)' "$RAW"
cast call --rpc-url "$RPC" "$REGISTRY" 'findOwner(string)(address)' "$NFC"
```

Observed:

```
raw hash       0x0e42af7a26b2034dcf0a8bcb968d5dfbd183059d65751b13c27d60375980cdb9
canonical hash 0x0aa3f3885ac1cb2cf476a3fe280713f0e53f614d493e506f53268598faf19771
raw available=false; owner=0x13D7...D951
NFC available=true; owner=0x0000000000000000000000000000000000000000
```

3. The indexer confirms the raw on-chain domain is noncanonical:

```graphql
{
  domains(where: {
    name: "e\u0301hca-1a016f8610f.eth",
    includeUnnormalized: true
  }) {
    name normalizedName isNormalized isReachable owner { id }
  }
}
```

`https://staging-graphql.ens.dev/` returns `normalizedName: "éhca-1a016f8610f.eth"`, `isNormalized: false`, `isReachable: true`, and the owner above. The normal dashboard owner query returns 5 domains; adding `includeUnnormalized: true` returns 10, including this token.

4. Open the raw route:\
   <https://app.ens.dev/e%CC%81hca-1a016f8610f.eth\\>
   Manager displays `éhca-1a016f8610f.eth doesn't exist`. [Screenshot](broken://pages/25221aa94483d7928291e1a26c11abdd4748ea5e)
5. Open the visually identical canonical route:\
   <https://app.ens.dev/register/%C3%A9hca-1a016f8610f.eth\\>
   Manager displays registration pricing and `Connect to register`. [Screenshot](broken://pages/c3b56b34addb03e1740ab0bfe9abf81bdecb9f0c)

For a fresh attack, the attacker registers the NFC form first and sends the victim the NFD URL. The victim's independent NFD registration succeeds while normalized resolution selects the attacker-owned NFC identity.
