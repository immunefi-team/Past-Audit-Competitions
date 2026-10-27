# Attackathon \_ Fuel Network 33488 - \[Smart Contract - Medium] Insecure implementation of StorageMap c

Submitted on Sun Jul 21 2024 20:51:55 GMT-0400 (Atlantic Standard Time) by @jecikpo for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33488

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Security Best Practices
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Currently `StorageMap` implementation could lead to unintended storage slot overwrites if misused by the developer. `StorageMap` uses hashing mechanism (through the usage of `field_id` provided by the developer) to create unique storage slot values by hashing the key through the `field_id`. Nothing however prevents from reusing the `field_id` in a different `StorageMap` instance and hence having the two `StorageMap` instances collide.

## Vulnerability Details

Sway offer a `StorageMap` facility for secure key-value data writing into contract's storage. It uses the `field_id` to hash the key and create a unique storage slot value.

This works correctly when the developer ensures the uniqueness of the `field_id`. In the future however when the ecosystem grows and dapps are composed of multiple different libraries written by different parties, this will become harder to ensure.

## Impact Details

While this is not strictly a bug in the code, it is a faulty library design which might have consequences for future projects. In the event of `StorageMap` collision the impact can be severe as the storage overwritten could result in users assets stolen, as the `StorageMap` shall be used to store the state of user's interaction with the dapp.

An example would be a vault which uses `StorageMap` to record users's deposits. Another overlapping `StorageMap` could then be leveraged to create mallicious entries which would then be read by the first `StorageMap` and hence fake deposits could be created.

The security of `StorageMap` relies heavily on correctness of the implementation and as Fuel demonstrated strong intent of enforcing correctness of implementation (e.g. through the CEI verification during compile time), I think it is correct to report this under High severity.

## Solution Proposal

I created a draft of `SecureStorageMap` implementation which is based on how Solidity creates unique slots for multiple `mappings`. It's description along with code can be found here for reference:

<https://gist.github.com/jecikpo/f4508ab48ef91ca42866fcc1645998a4>

## References

The current implementation: <https://github.com/FuelLabs/sway/blob/4adae25aa7e5b954c4c87bd5c683a79c3373f540/sway-lib-std/src/storage/storage\\_map.sw#L54>

## Proof of concept

## Proof of Concept

PoC along with description can be found here: <https://gist.github.com/jecikpo/5fc57811142c96f85e83037386bfade5>
