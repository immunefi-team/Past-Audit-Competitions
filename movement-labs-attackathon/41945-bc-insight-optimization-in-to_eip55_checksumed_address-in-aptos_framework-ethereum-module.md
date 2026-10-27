# #41945 \[BC-Insight] Optimization in \`to\_eip55\_checksumed\_address()\` in \`aptos\_framework::ethereum::()\` module

**Submitted on Mar 19th 2025 at 14:31:59 UTC by @p4y4b13 for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41945
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**

## Description

## Brief/Intro

The function to\_eip55\_checksumed\_address() converts Ethereum address to EIP-55 checksummed format. For this it will follow the below steps

1. First converts the address to the lowercase
2. Compute's keccak-256 hash of the lowercase address
3. Uppercase Letters (a-f) -> if their corresponding hash character is >= 8

The existing Move implementation of EIP-55 checksum extraction logic is correct but can be optimized.

## Vulnerability Details

The EIP-55 checksum process determines whether each letter in an Ethereum address should be uppercase or lowercase based on the Keccak hash of the lowercase address. Each hexadecimal character (nibble) in the address is compared against the corresponding nibble from the hash.

In the Move implementation, the nibble extraction logic is written as:

```rust
let hash_item = *vector::borrow(&hash, index / 2);
if ((hash_item >> ((4 * (1 - (index % 2))) as u8)) & 0xF >= 8) {
    vector::push_back(&mut output, item - 32);
} else {
    vector::push_back(&mut output, item);
}
```

This correctly extracts the left nibble for even indices and the right nibble for odd indices. However, the expression 4 \* (1 - (index % 2)) is unnecessarily complex.

Example with Values :

Assume the lowercase address: fb6916...

Keccak hash (hex): 3f3a5f...

Processing index = 0 ('f') :

* index / 2 = 0, so hash\_item = 0x3f (0011 1111 in binary).
* index % 2 = 0, so 4 \* (1 - 0) = 4 → shift right by 4 bits.
* Extract left nibble: 0x3 = 0011.
* 3 < 8, so keep f lowercase

Processing index = 1 ('b')

* index / 2 = 0, so hash\_item = 0x3f (0011 1111).
* index % 2 = 1, so 4 \* (1 - 1) = 0 → shift right by 0 bits.
* Extract right nibble: 0xf = 1111.
* 15 >= 8, so uppercase b → B

This logic is correct, but could be written more clearly.

## Impact Details

The impact is not critical, but simplifying the nibble extraction logic improves code readability and optimizes gas usage by reducing unnecessary complexity.

## References

[Link](https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/ethereum.move#L90-L110)

```rust
public fun to_eip55_checksumed_address(ethereum_address: &vector<u8>): vector<u8> {
        assert!(vector::length(ethereum_address) == 40, 0);
        let lowercase = to_lowercase(ethereum_address);
        let hash = keccak256(lowercase);
        let output = vector::empty<u8>();

        for (index in 0..40) {
            let item = *vector::borrow(ethereum_address, index);
            if (item >= ASCII_A_LOWERCASE && item <= ASCII_F_LOWERCASE) {
                let hash_item = *vector::borrow(&hash, index / 2);
                if ((hash_item >> ((4 * (1 - (index % 2))) as u8)) & 0xF >= 8) {
                    vector::push_back(&mut output, item - 32);
                } else {
                    vector::push_back(&mut output, item);
                }
            } else {
                vector::push_back(&mut output, item);
            }
        };
        output
    }
```

## Recommendation:

Replace the nibble extraction logic in `to_eip55_checksumed_address()` with a simpler approach

```rust
let hash_item = *vector::borrow(&hash, index / 2);
let hash_nibble = if index % 2 == 0 {
    (hash_item >> 4) & 0xF  // Extract left nibble for even indices
} else {
    hash_item & 0xF  // Extract right nibble for odd indices
};

if hash_nibble >= 8 {
    vector::push_back(&mut output, item - 32);
} else {
    vector::push_back(&mut output, item);
}
```

## Proof of Concept

## Proof of Concept

Paste the following code in `ethereum.move` file and run the following test using the below command

```rust
aptos move test --filter test_valid_eip55_checksum_poc
```

PoC :

```rust
    public fun to_eip55_checksumed_address_2(ethereum_address: &vector<u8>): vector<u8> {
        assert!(vector::length(ethereum_address) == 40, 0);
        let lowercase = to_lowercase(ethereum_address);
        let hash = keccak256(lowercase);
        let output = vector::empty<u8>();

        for (index in 0..40) {
            let item = *vector::borrow(ethereum_address, index);
            if (item >= ASCII_A_LOWERCASE && item <= ASCII_F_LOWERCASE) {
                let hash_item = *vector::borrow(&hash, index / 2);
                let hash_nibble = if (index % 2 == 0) {
                    (hash_item >> 4) & 0xF  // Extract left nibble for even indices
                } else {
                    hash_item & 0xF  // Extract right nibble for odd indices
                };

                if (hash_nibble >= 8) {
                    vector::push_back(&mut output, item - 32);
                } else {
                    vector::push_back(&mut output, item);
                }
            } else {
                vector::push_back(&mut output, item);
            }
        };
        output
    }

    public fun assert_eip55_2(ethereum_address: &vector<u8>) {
        let eip55 = to_eip55_checksumed_address_2(ethereum_address);
        let len = vector::length(&eip55);
        for (index in 0..len) {
            assert!(vector::borrow(&eip55, index) == vector::borrow(ethereum_address, index), 0);
        };
    }

    #[test]
    fun test_valid_eip55_checksum_poc() {
        // testing using the previous eip55_checksumed_address logic
        assert_eip55(&valid_eip55());

        // testing after the updated eip55_checksumed_address logic. both will succedd.
        assert_eip55_2(&valid_eip55());
    }
```

output of the test :

```rust
Running Move unit tests
[ PASS    ] 0x1::ethereum::test_valid_eip55_checksum_poc
Test result: OK. Total tests: 1; passed: 1; failed: 0
{
  "Result": "Success"
}
```
