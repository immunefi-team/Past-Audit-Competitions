# #37545 \[BC-Medium] Deposits with a lock\_time of 16 cannot be processed

**Submitted on Dec 8th 2024 at 07:54:54 UTC by @PaiMei\_and\_Gandalf for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37545
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/sbtc>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

Reclaim scripts that want a lock\_time of 16 blocks must be created as the opcode `OP_PUSHNUM_16` followed by opcode `OP_CSV` and followed by the rest of the reclaim script (whatever it may be). It turns out trying to parse a reclaim script with this format throws an error, and deposits with this format cannot be processed by `sBTC`.

## Vulnerability Details

The parsing of a reclaim script in file `sbtc/src/deposits.rs` matches the script with three patterns, which are the only ones valid for the `sBTC` project.

```rust
    pub fn parse(reclaim_script: &ScriptBuf) -> Result<Self, Error> {
        let (lock_time, script) = match reclaim_script.as_bytes() {
            // These first two branches check for the case when the script
            // is written with as few bytes as possible (called minimal
            // CScriptNum format or something like that).
            [0, OP_CSV, script @ ..] => (0, script),
            // This catches numbers 1-16 and -1. Negative numbers are
            // invalid for OP_CHECKSEQUENCEVERIFY, but we filter them out
            // later in `ReclaimScriptInputs::try_new`.
            [n, OP_CSV, script @ ..]
                if OP_PUSHNUM_NEG1 == *n || (OP_PUSHNUM_1..OP_PUSHNUM_16).contains(n) =>
            {
                (*n as i64 - OP_PUSHNUM_1 as i64 + 1, script)
            }
            // Numbers in bitcoin script are typically only 4 bytes (with a
            // range from -2**31+1 to 2**31-1), unless we are working with
            // OP_CSV or OP_CLTV, where 5-byte numbers are acceptable (with
            // a range of 0 to 2**39-1). See the following for how the code
            // works in bitcoin-core:
            // https://github.com/bitcoin/bitcoin/blob/v27.1/src/script/interpreter.cpp#L531-L573
            // That said, we only accepts 4-byte unsigned integers, and we
            // check that below.
            [n, rest @ ..] if *n <= 5 && rest.get(*n as usize) == Some(&OP_CSV) => {
                // We know the error and panic paths cannot happen because
                // of the above `if` check.
                let (script_num, [OP_CSV, script @ ..]) = rest.split_at(*n as usize) else {
                    return Err(Error::InvalidReclaimScript);
                };
                (read_scriptint(script_num, 5)?, script)
            }
            _ => return Err(Error::InvalidReclaimScript),
        };

        let lock_time =
            u32::try_from(lock_time).map_err(|_| Error::InvalidReclaimScriptLockTime(lock_time))?;

        let script = ScriptBuf::from_bytes(script.to_vec());
        ReclaimScriptInputs::try_new(lock_time, script)
    }
```

We can see in the second branch of the `match` statement, this comment:

> // This catches numbers 1-16 and -1

```rust
...
    [n, OP_CSV, script @ ..]
        if OP_PUSHNUM_NEG1 == *n || (OP_PUSHNUM_1..OP_PUSHNUM_16).contains(n) =>
    {
        (*n as i64 - OP_PUSHNUM_1 as i64 + 1, script)
    }
...
```

The problem here is `OP_PUSHNUM_1..OP_PUSHNUM_16` does not include `OP_PUSHNUM_16` in the range, so trying to parse a script with this format will throw an `InvalidReclaimScript` error.

The fix for this bug is to explicitly include the last element in the range.

```diff
...
    [n, OP_CSV, script @ ..]
-        if OP_PUSHNUM_NEG1 == *n || (OP_PUSHNUM_1..OP_PUSHNUM_16).contains(n) =>
+        if OP_PUSHNUM_NEG1 == *n || (OP_PUSHNUM_1..=OP_PUSHNUM_16).contains(n) =>
    {
        (*n as i64 - OP_PUSHNUM_1 as i64 + 1, script)
    }
...
```

The fact that using a non minimal reclaim script to try to accomplish the same result (having a `lock_time` of 16 blocks) would parse correctly but would recreate a different reclaim script than the one parsed, would make the function `validate_tx` also throw an error when trying to validate the transaction:

```rust
    pub fn validate_tx(&self, tx: &Transaction) -> Result<DepositInfo, Error> {
...
        // Validate that the deposit and reclaim scripts in the request
        // match the expected formats for deposit transactions.
        let deposit = DepositScriptInputs::parse(&self.deposit_script)?;
        let reclaim = ReclaimScriptInputs::parse(&self.reclaim_script)?;
        // Okay, the deposit and reclaim scripts are valid. Now make sure
        // that the ScriptPubKey in the transaction matches the one implied
        // by the given scripts. So now create the expected ScriptPubKey.
        let deposit_script = deposit.deposit_script();
        let reclaim_script = reclaim.reclaim_script();

        debug_assert_eq!(deposit_script, self.deposit_script);
        debug_assert_eq!(reclaim_script, self.reclaim_script);

        let expected_script_pubkey =
            to_script_pubkey(deposit_script.clone(), reclaim_script.clone());
        // Check that the expected scriptPubkey matches the actual public
        // key of our parsed UTXO.
        if expected_script_pubkey != tx_out.script_pubkey {
            return Err(Error::UtxoScriptPubKeyMismatch(self.outpoint));
        }
...
    }
```

Because `&self.reclaim_script` and `let reclaim_script = reclaim.reclaim_script();` will end up being different, and then `expected_script_pubkey` will be different than `tx_out.script_pubkey` and the validation will throw a `UtxoScriptPubKeyMismatch` error.

In conclusion, no script with a `lock_time` of 16 can be used currently in `sBTC`.

## Impact Details

The impact is High because users will lose the fees of the Bitcoin deposit, and the later reclaim of funds for using a valid reclaim script that sBTC simply does not recognize as valid. Also, their funds will be unavailable for them to use for 16 blocks.

## References

<https://github.com/stacks-network/sbtc/blob/immunefi\\_attackaton\\_0.9/sbtc/src/deposits.rs#L446-L485> <https://github.com/stacks-network/sbtc/blob/immunefi\\_attackaton\\_0.9/sbtc/src/deposits.rs#L126-L172>

## Proof of Concept

## Proof of Concept

We added the following unit test to file `sbtc/src/deposits.rs`.

```rust
    #[test]
    fn OP_PUSHNUM_16_disallowed() {
        let lock_time = 16;

        // The first script is simply the OP_PUSHNUM_16 opcode followed by the OP_CSV opcode
		// It should be parsed correctly, but instead throws an error
        let reclaim_script = ScriptBuf::from_bytes(vec![0x60, 0xB2]);
        let error = ReclaimScriptInputs::parse(&reclaim_script).unwrap_err();

        assert!(matches!(error, Error::InvalidReclaimScript));

        // The second script is 1, followed by the number 16 and followed by the OP_CSV opcode
        // This is a non minimal representation that produces the same script, a lock_time of 16
        // In this case the script is parsed correctly, but when reconstructed it is not equal
        // to reclaim_script2, but to reclaim_script (the one that uses OP_PUSHNUM_16)
        let reclaim_script2 = ScriptBuf::from_bytes(vec![0x01, 0x10, 0xB2]);
        let extracts = ReclaimScriptInputs::parse(&reclaim_script2).unwrap();

        assert_ne!(extracts.reclaim_script(), reclaim_script2);
        assert_eq!(extracts.reclaim_script(), reclaim_script);
    }
```
