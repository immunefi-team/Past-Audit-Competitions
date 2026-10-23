# #40806 \[BC-High] Users can submit deposits containing large \`reclaim\_scripts\` to DoS Emily and Signers

**Submitted on Mar 4th 2025 at 08:26:08 UTC by @f4lc0n for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #40806
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

Users initiate deposits by executing BTC transactions, which contain the hashes of `deposit_script` and `reclaim_script`. The benefit of using hashing is that the size of the deposit BTC transaction is fixed, no matter how big the `deposit_script` and `reclaim_script` are. This saves gas fees for deposit BTC transactions. And the Emily and Signers local databases will store `deposit_script` and `reclaim_script`.

However, the problem now is that Emily and Signers do not limit the size of the `reclaim_script`, so the attacker can submit large `reclaim_script`s (the size of the deposit BTC transaction will not change) to exhaust Emily and Signers' database.

## Vulnerability Details

The `sbtc/src/deposits.rs::ReclaimScriptInputs::parse` function code is as follows. It does not limit the size of the `reclaim_script`.

````rust
    /// Parse the reclaim script for the lock time.
    ///
    /// The goal of this function is to make sure that there are no
    /// surprises in the reclaim script. These scripts are conceptually
    /// very simple and are their format is
    /// ```text
    ///  <locked-time> OP_CHECKSEQUENCEVERIFY <rest-of-reclaim-script>
    /// ```
    /// This function extracts the <locked-time> from the script. If the
    /// script does not start with <locked-time> OP_CHECKSEQUENCEVERIFY
    /// then we return an error.
    ///
    /// See https://github.com/stacks-network/sbtc/issues/30 for the
    /// expected format of the reclaim script. And see BIP-0112 for
    /// the details and input conditions of OP_CHECKSEQUENCEVERIFY:
    /// https://github.com/bitcoin/bips/blob/812907c2b00b92ee31e2b638622a4fe14a428aee/bip-0112.mediawiki#summary
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
                if OP_PUSHNUM_NEG1 == *n || (OP_PUSHNUM_1..=OP_PUSHNUM_16).contains(n) =>
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
````

## Impact Details

Attacker can make Emily and Signers unavailable due to insufficient database space. All sBTC withdrawals will then be frozen and the deposits feature will be unavailable.

## References

None

## Proof of Concept

## Proof of Concept

* Base on: <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* Auditor wallet address: `ST2BEV097EV2R9ZMFRMRT904QB5RFYMA0683TC111`
* Auditor wallet mnemonics: `spawn knee orchard patrol merge forget dust position daring short bridge elevator attitude leopard opera appear auction limit magic hover tunnel museum quantum manual`

1. Patch `docker/stacks/stacks-regtest-miner.toml`. Give the auditor address some STX for testing.

   ```diff
    [[ustx_balance]]
    address = "ST3497E9JFQ7KB9VEHAZRWYKF3296WQZEXBPXG193" # Demo principal
    amount = 10000000000000000
    
   +[[ustx_balance]]
   +address = "ST2BEV097EV2R9ZMFRMRT904QB5RFYMA0683TC111" # Auditor principal
   +amount = 10000000000000000
   ```
2. Add [this code](https://gist.github.com/al-f4lc0n/25bd18b41484fab5e3a121df236bbc7d) to `signer/src/bin/pocinit.rs`. On the basis of `./signers.sh demo` command, it also deposited some sBTC to the auditor address for testing
3. Add `pocinit` bin to `signer/Cargo.toml`

   ```diff
   + [[bin]]
   + name = "pocinit"
   + path = "src/bin/pocinit.rs"
   ```
4. Add [this code](https://gist.github.com/al-f4lc0n/e2218fc8810a7e6e7a3abdf30f315955) to `signer/src/bin/pocii2.rs`.
5. Add `pocii2` bin to `signer/Cargo.toml`

   ```diff
   + [[bin]]
   + name = "pocii2"
   + path = "src/bin/pocii2.rs"
   ```
6. Run `devenv` and run `pocinit`

   ```sh
   make devenv-up
   cargo run -p signer --bin pocinit
   ```
7. Run PoC. This PoC will execute a BTC transaction containing 100 attack deposits. Together they will take up 19.43 MB of database size

   ```sh
   cargo run -p signer --bin pocii2
   ```
