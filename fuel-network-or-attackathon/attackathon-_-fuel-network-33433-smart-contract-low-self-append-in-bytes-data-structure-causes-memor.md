# Attackathon \_ Fuel Network 33433 - \[Smart Contract - Low] Self-append in Bytes data structure causes

Submitted on Sat Jul 20 2024 10:56:10 GMT-0400 (Atlantic Standard Time) by @Solosync6 for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33433

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A vulnerability exists in the bytes data structure where appending the same bytes object to itself can lead to memory corruption and potential exploits. (Note that this issue also exists in other primitives, such as Vec, but I'm using Bytes for this issue). This operation clears out the bytes object and hence becomes an easy attack vector for DOS attacks for protocols using the append function.

## Vulnerability Details

The issue arises when the Bytes data structure allows an object to append itself. This operation does not properly handle the internal pointers and capacity, leading to potential memory corruption.

Here's an snippet of the problematic code in sway::sway-lib-std::bytes.sw :

```rust
    pub fn append(ref mut self, ref mut other: self) { 
        let other_len = other.len(); //@audit no check if other != self
        if other_len == 0 {
            return
        };

        // optimization for when starting with empty bytes and appending to it
        if self.len == 0 {
            self = other;
            other.clear();
            return;
        };

        let both_len = self.len + other_len;
        let other_start = self.len;

        // reallocate with combined capacity, write `other`, set buffer capacity
        if self.buf.capacity() < both_len {
            let new_slice = raw_slice::from_parts::<u8>(
                realloc_bytes(self.buf.ptr(), self.buf.capacity(), both_len),
                both_len,
            );
            self.buf = RawBytes::from(new_slice);
        }

        let new_ptr = self.buf.ptr().add_uint_offset(other_start);
        other.ptr().copy_bytes_to(new_ptr, other_len);

        // set capacity and length
        self.len = both_len;

        // clear `other`
        other.clear(); //@audit clears all the elements 
    }
```

As is highlighted above, appending bytes object to itself clears the object completely.

## Impact Details

Imagine a scenario where a smart contract maintains a Bytes instance to store a log of transactions. Each transaction is added to the log using the `append` function.

Now, due to a programming error or a malicious attack, the contract mistakenly calls append on the log itself. This could potentially wipe out the entire transaction history, becoming an potential DOS attack vector.

Self-appending could lead to an empty Bytes instance disrupting key storage of a protocol. Rating it as MEDIUM because instances where this could happen are relatively rare - however, this needs to be fixed as the potential outcome is memory corruption that can have unexpected consequences.

## References

<https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/sway-lib-std/src/bytes.sw#L695>

### Recommendation

Consider adding a check that the self pointer and other pointer are different before appending. If you want to support self-append scenario, consider cloning the `other` object before appending.

When clearing memory, in general, please be mindful of self referencing across the codebase.

## Proof of concept

## Proof of Concept

Copy the following test to bytes.sw and run the following inside the sway-lib-std folder:

> forc test --logs --filter-exact test\_self\_append

\#\[test] fn test\_self\_append() { let mut bytes = Bytes::new(); bytes.push(1); bytes.push(2); bytes.push(3);

```
// Attempt to append bytes to itself
bytes.append(bytes);

// What should the result be? 
// Ideal: [1, 2, 3, 1, 2, 3]
// Actual: Undefined behavior, possible corruption

// Print or assert the results
let mut i = 0;
while i < bytes.len() {
    let value = bytes.get(i).unwrap();
    __log(value);
    i += 1;
}

assert(bytes.len() == 0);    
```

}
