# #38671 \[BC-Insight] Signer key rotation is not possible due to deadlock between submitting key rotat

**Submitted on Jan 9th 2025 at 09:32:27 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38671
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

Rotating a signer's key is currently impossible since signers build their state machine for DKG based on the last signer set they have in their database. This set, however, is only set AFTER a DKG was successful and a corresponding transaction was sent to Stacks. These two actions basically deadlock each other, preventing key rotations overall.

## Finding Description

In `transaction_signer::handle_wsts_message`, we see the signing set used for DKG is retrieved from the signer's database:

```rs
WstsNetMessage::DkgBegin(_) => {
    tracing::info!("handling DkgBegin");

    if !chain_tip_report.sender_is_coordinator {
        tracing::warn!("received coordinator message from non-coordinator signer");
        return Ok(());
    }

    let signer_public_keys = self.get_signer_public_keys(bitcoin_chain_tip).await?; // <------
    // [...]
}
```

This defaults to the bootstrap signer set from the config if there is no other `rotate-keys` transaction in the signer's database. This shows us the first requirement, namely that we need to have a `rotate-keys` transaction confirmed on Stacks, containing a NEW signer set.

Now in order to submit such a transaction, the coordinator must get signatures for a `ContractCall::RotateKeysV1` contract call, which get validated by the singers:

```rs
async fn validate<C>(&self, ctx: &C, req_ctx: &ReqContext) -> Result<(), Error>
where
    C: Context + Send + Sync,
{
    // [...]
    let Some(latest_dkg) = db.get_latest_encrypted_dkg_shares().await? else {
        return Err(Error::NoDkgShares);
    };
    let latest_public_key = latest_dkg
        .signer_set_public_keys
        .into_iter()
        .collect::<BTreeSet<_>>();
    if self.new_keys != latest_public_key {
        return Err(RotateKeysErrorMsg::SignerSetMismatch.into_error(req_ctx, self));
    }
    // [...]
}
```

We see, that a requirement here, is that the new signer set must match the latest signer set we have in our database. Now this signer set is ONLY set after a successful DKG.

Putting these together, we see that we have a logical deadlock making it impossible for a key rotation to happen

## Severity

I see this as Insight, since key rotations are technically disabled currently. If they should be usable right now, this would be medium in my opinion since it breaks intended functionality.

## Mitigation

In order to properly rotate a key, we need additional functionality for signers to add a temporary "proposed" signer set to their storage. This would then need to be retrieved when a DKG is triggered. This should probably be done manually by each individual signer in order to guarantee a quorum of signers agrees with the change (a new signer set can only be adopted, if a quorum of the old signer set agrees on it).

Note: Do NOT allow the coordinator to specify the new set since this would potentially allow them to take over the system.

## Proof of Concept

I was asked to submit this without a PoC by djordon
