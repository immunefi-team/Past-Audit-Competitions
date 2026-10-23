# Attackathon \_ Fuel Network 32612 - \[Smart Contract - Low] Lack of slot hashing at adminsw can cause

Submitted on Thu Jun 27 2024 11:17:22 GMT-0400 (Atlantic Standard Time) by @jecikpo for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32612

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Unauthorized minting of NFTs
* Access Control bypass
* Permanent freezing of NFTs
* Permanent freezing of funds

## Description

## Brief/Intro

The `StorageKey` is used at `sway-libs/libs/src/admin.sw` and it allows to place various values under different slots in storage. Usage of `Identity` as key in `StorageKey` and lack of any hashing of the key can cause predictable storage slots and hence storage slot collisions in case `Identity` is re-used by 3rd party libs to store anything, which can be easily abused based on specific dapp implementations.

## Vulnerability Details

`StorageKey` library allows to store data on the contract storage. `StorageKey` object contains three members:

`slot`: \[b256] - The assigned location in storage for the new StorageKey. `offset`: \[u64] - The assigned offset based on the data structure T for the new `StorageKey`. `field_id`: \[b256] - A unique identifier for the new `StorageKey`. The problem is that in the `new()` function the slot is assigned and it is not changed in any way:

```rust
pub fn new(slot: b256, offset: u64, field_id: b256) -> Self {
        Self {
            slot,
            offset,
            field_id,
        }
    }
```

Also the `write()` function takes the slot directly:

```rust
    pub fn write(self, value: T) {
        write(self.slot(), self.offset(), value);
    }
```

The issue here is that if the slot is taken exactly as is from the `Identity`. If there is any other 3rd party library which also puts information on the storage using some `Identity` as key, it can easily be used to overwrite the existing admin account information and hence breaking the access control mechanisms.

Here we can see how a new admin account is added in the `add_admin()`:

```rust
    let admin_key = StorageKey::<Identity>::new(new_admin.bits(), 0, new_admin.bits());
    admin_key.write(new_admin);
```

The expectation here would be that the slot and `field_id` would be hashed together to form the actual storage slot location. This way, if the `field_id` is hardcoded, the impact of storage writing/reading is contained within the contract in a well defined scope. While `StorageKey` is not using `field_id` by design, it would be recommended to make use of `StorageMap` to hold the admin accounts information in the storage.

Switching from `StorageKey` to `StorageMap` makes the implementation in line with `sway-libs/libs/src/ownership.sw` which uses a hash digest of `sha256("owner")` as the holding slot of the owner.

The decision to use `StorageKey` instead of `StorageMap` was based on lower gas consumption of `StorageKey` vs `StorageMap`. This however compromised security based on the above description. Gas usage normally should not be considered for access control guarded function because they are mostly rarely called and only by the protocol team's approved accounts.

## Impact Details

The impact of this is highly dependent on the actual dapp implementation, however as this is a low level function it could easily be used to overwrite access to minting Coins or NFTs issuance functions. Usage of admin typically controls also the upgrade process of a protocol and so could lead to a destruction of a protocol if used by a malicious user (hence the permanent freeze of assets impact listed)

We can imagine that there is another 3rd party library which uses `Identity` to store user data, a user could put there data that would be his own identity. `only_admin()` could then be reading that data and assume that the normal user is a real admin.

I chose the impact as critical, because the security of admin.sw implementation in the current form is depending on the 3rd party libraries implementations (so that they won't use `StorageKey` in that certain way) in the entire future ecosystem. and admin.sw could become a critical component of almost every dapp built on Fuel.

Regarding the likeliness of the abuse, in order to keep the current implementation secure, no other external library or code would use `Identity` as `slot` argument of the `StorageKey`, which is a requirement that would be hard to enforce.

## References

`StorageKey::new()` function: <https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-lib-core/src/storage.sw#L45C5-L51C6>

`StorageKey::write()` function: <https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-lib-std/src/storage/storage\\_key.sw#L56C5-L58C6>

`admin.sw` implementing `add_admin()` function: <https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/admin.sw#L37C8-L37C17>

## Proof of concept

## Proof of Concept

The following gist contains the PoC: <https://gist.github.com/jecikpo/9d68846ec310727caca86ce187444523>
