# #38030 \[BC-Insight] Coordinator can be crashed by signers on DKG

**Submitted on Dec 22nd 2024 at 18:10:47 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38030
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Summary

The `wsts` library uses insecure array accesses, making it possible for a malicious signer to send certain packets to a coordinator during a DKG which cause the coordinator to panic.

## Finding Description

I will put multiple instances of this in one issue since they all are only triggerable on DKG (which will not be triggered in the near future but this was explicitly mentioned by the team to still be desireable).

Instances:

1. `wsts::fire.rs#L580`
2. `wsts::fire.rs#L626`
3. `wsts::fire.rs#L628`

Further description:

1. Here we index `dkg_public_shares[bad_signer_id]`. If a signer manages to get a malicious `bad_signer_id` to this point, we will crash. This is the least likely case of the here mentioned cases but if a signer manages to have `bad_signer_id` not in `dkg_public_shares`, it is possible. While this is not easy, I wanted to mention it anyways for completeness
2. Here we index `dkg_public_shares[src_party_id]` where we get `src_party_id` by iterating over `dkg_private_shares.shares` of which the integrity is NOT checked so this is directly exploitable by the attack described later
3. Here we index `key_shares[key_id]` where `key_shares` is also taken from `dkg_private_shares.shares` making it a direct target for exploitation

### Exploiting 2) and 3)

In order to exploit this, a malicous signer needs to send a `BadPrivateShare` to the coordinator which contains malformed `shares` and adheres to the following restrictions:

* `tuple_proof` is valid

## Mitigation

Replace all array accesses in the whole `wsts` library using `[]` with `.get` or equivalent methods and handle errors accordingly.

## Proof of Concept

## PoC

@djordon mentioned on discord that I don't need to provide a PoC for these since this report is supposed to be marked as `Insight` anyways.
