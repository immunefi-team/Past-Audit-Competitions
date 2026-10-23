# Attackathon \_ Fuel Network 32924 - \[Smart Contract - Insight] sways legacy storage namespacing is br

Submitted on Sun Jul 07 2024 15:22:13 GMT-0400 (Atlantic Standard Time) by @cyberthirst for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32924

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

Sway allows to namespace storage to avoid collisions. It does so by adding salt to the computation of the storage slot of the given storage variable. However, at least in certain cases, the salt is not used in the computation, and thus, the contracts can be vulnerable to storage collisions, especially when the proxy pattern is employed.

## Vulnerability Details

In the compiler, the storage key is calculated as:

```
/// Hands out storage keys using storage field names or an existing key.
/// Basically returns sha256("storage::<storage_namespace_name1>::<storage_namespace_name2>.<storage_field_name>")
/// or key if defined.
pub(super) fn get_storage_key(storage_field_names: Vec<String>, key: Option<U256>) -> Bytes32 {
    if let Some(key) = key {
        return key.to_be_bytes().into();
    }

    Hasher::hash(get_storage_key_string(storage_field_names))
}
```

The `storage_field_names` includes the respective namespaces and the fields. However, in the case of namespacing via:

```
#[namespace(example_namespace)]
storage {
```

the namespace name is not provided at all in the `vec`. Therefore, the storage slot is computed without salt.

This is most likely a regression introduced in the recent PR, which added a new namespacing variant (see the link in References).

In the PoC, the proxy uses the `foo` namespace to avoid storage collisions with important storage variables, and the implementation doesn't use any namespace. Yet, both end up having the same storage addresses.

If the alternative variant:

```
storage {
namespace {
...
}
}
```

is used, the salts are propagated correctly.

## Impact Details

The vulnerable namespacing is said to be deprecated by the compiler. However, it is still accessible without any flags, and it is also `the only` namespacing variant documented in the Sway docs.

The PoC demonstrated that both Proxy and Implementation use the same storage addresses. As such, the implementation can unintentionally rewrite the address of the implementation contract and also the owner. As such, the proxy can become unusable because it might point to an invalid implementation contract (and it can't be updated because the owner was also rewritten). Therefore, the funds are blocked forever because the proxy doesn't have implementation.

But because this is a compiler bug, any user contract that relies on namespacing to avoid storage collision is potentially vulnerable.

## References

Sway namespacing docs: <https://docs.fuel.network/docs/sway/advanced/advanced\\_storage/#storage-namespace> Namespacing PR: <https://github.com/FuelLabs/sway/commit/e30497caaae37683011245d45ec4fc84b368f3ae#diff-bfae318752d54fae537a316de96038bab0ff0e1eebe44643d63ce9bcca784de7>

## Proof of concept

## Proof of Concept

The whole project is accessible as .zip from the link at the bottom. Here, we describe the main points.

The main things to pay attention to are the storage declarations in proxy.sw and impl.sw.

In proxy we have

```
#[namespace(foo)]
storage {
```

and in impl we have the same declarations but without the namespace:

```
storage {
```

Now, if the `poc.sh` bash script is run, it loads the `storage_slots.json` of both the contracts and compares their contents. If the contents are identical, it outputs `COLLISION FOUND`. And this, indeed, is the case.

The script just builds and loads json files:

```
#!/bin/bash

# Remove existing build artifacts
rm -rf implementation/out proxy/out

# Build both projects
cd implementation && forc build && cd ..
cd proxy && forc build && cd ..

# Load contents of storage_slots.json from both projects
implementation_storage=$(cat implementation/out/debug/implementation-storage_slots.json)
proxy_storage=$(cat proxy/out/debug/proxy-storage_slots.json)

# Compare the contents
if [ "$implementation_storage" = "$proxy_storage" ]; then
    echo "COLLISION FOUND"
else
    echo "NO COLLISION FOUND"
fi
```

We attach the whole project through Google Drive. Once downloaded and unzipped, only run `chmod +x poc.sh` and `./poc.sh`.

Google Drive link: <https://drive.google.com/file/d/1lPRe5eQKxULZkzHBhNWcFo-\\_DrVdGkPv/view?usp=sharing>
