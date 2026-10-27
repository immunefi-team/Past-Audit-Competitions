# #42750 \[BC-Insight] Subtraction overflow risk in WSTS FIRE coordinator

**Submitted on Mar 25th 2025 at 17:59:39 UTC by @Blobism for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #42750
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/blob/immunefi\\_attackaton\\_1.0/Cargo.toml#L31>
* **Impacts:**

## Description

## Brief/Intro

There is a risk of a subtraction overflow in the WSTS FIRE coordinator when handling a SigShareGather timeout. The risk is that a sufficient number of malicious signers in a signing round could force the round to keep getting repeated if the subtraction overflow is accomplished.

## Vulnerability Details

The subtraction overflow risk is shown below, from `src/state_machine/coordinator/fire.rs`. Given the complexity of the logic here, security best practice would be to make the subtraction a `saturating_sub`.

```rust
let num_malicious_keys: u32 = self
    .malicious_signer_ids
    .iter()
    .map(|signer_id| self.config.signer_key_ids[signer_id].len() as u32)
    .sum();

if self.config.num_keys - num_malicious_keys < self.config.threshold { // <--- subtraction overflow risk
    error!("Insufficient non-malicious signers, unable to continue");
    let mal = self.malicious_signer_ids.iter().copied().collect();
    return Ok((
        None,
        Some(OperationResult::SignError(
            SignError::InsufficientSigners(mal),
        )),
    ));
}
```

## Impact Details

I am submitting this as an insight, as there is plausible risk of malicious signers forcing repeated signing rounds. I have not fully investigated if the subtraction overflow can be accomplished in practice. Regardless, security best practice is to make this subtraction a `saturating_sub`.

## References

<https://github.com/Trust-Machines/wsts/tree/v13.0.1>

## Proof of Concept

I have not confirmed that a subtraction overflow can be accomplished. This report is simply a security best practice insight. I can conduct a deeper investigation if needed.
