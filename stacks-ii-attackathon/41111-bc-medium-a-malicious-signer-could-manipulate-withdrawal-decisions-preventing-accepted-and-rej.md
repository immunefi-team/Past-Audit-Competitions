# #41111 \[BC-Medium] A malicious signer could manipulate withdrawal decisions preventing accepted and rejected withdrawals from getting confirmed on Stacks chain

**Submitted on Mar 11th 2025 at 08:28:48 UTC by @ZoA for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #41111
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

For every sBTC withdrawal transaction, there happens a Stacks transaction to either accept or reject the withdrawal.\
These Stacks confirmation transactions pass `signer-bitmap` as an argument that represents whether each signer has accepted the withdrawal request or not.

A malicious signer could dispatch both `true` and `false` decisions at the same time to make other signers have different perspective of whether the malicious signer had accepted the request or not.

Its root cause might be because every signer only accepts the first decision from signers ignoring other decisions.

## Vulnerability Details

```rust
    // 10. That the signer bitmap matches the bitmap formed from our
    //     records.
    let votes = db
        .get_withdrawal_request_signer_votes(&self.id, &req_ctx.aggregate_key)
        .await?;
    if self.signer_bitmap != BitArray::from(votes) {
        return Err(WithdrawalErrorMsg::BitmapMismatch.into_error(req_ctx, self));
    }
```

For Stacks transaction signature request of `AcceptWithdrawalV1` and `RejectWithdrawalV1` contract call transactions, they validate `signer_bitmap` from received transaction payload against the data they have in their database, and reject the signature request when the data is different.

When a signer receives a withdrawal signer decision, they store it into the db, but only accepts the first one, as implemented in `write_withdrawal_signer_decision` of `postgres.rs`, where it does not do anything when conflict happens.

From the attacker's perspective, above types of Stacks transactions can not be executed if 3 or more signers have different decisions data in their database from other signers. So the goal of the attacker could be having half signers have true decision while other half have false decision.

A malicious signer could easily accomplish this by sending both true and false decisions at the same time, and by nature of the network, the order of packets received by each signer would be different.\
Or, in a more advanced way, the malicious signer could make a little changes to p2p `broadcast` function to send true decision to half of connected peers and false decision to other half of peers.

## Impact Details

Technically, a malicious signer could manipulate the decision of every withdrawal request.

* For accepted withdrawals, users will have BTC settled in Bitcoin, but the request wouldn't be never confirmed on Stacks chain, leaving locked tokens.
* For rejected withdrawals, the rejection transaction can't be executed on Stacks network, so the user can't get their sBTC tokens back, which is loss of funds for users.

## References

Codebase of `sBTC` is referred to discover the issue.

## Link to Proof of Concept

<https://gist.github.com/zoasec/03bc4c3a2256116754802e5c89a89066>

## Proof of Concept

## Proof of Concept

I've attached the git patch for changes to the protocol, which includes:

* Malicious signer configuration and making `sbtc-signer-1` a malicious one
* When `sbtc-signer-1` receives a withdrawal request, it broadcasts both `true` and `false` decisions.
* Modification in `event_loop.rs` to simulate the different order of receiving messages. This is because on local machine, all packets are received in the order of they are broadcasted, as there is no delay in local machine.
* Slight modification of `signers.sh` to add withdrawal feature.
* Some loggings.

After rebuilding the project, start the dev-env, and the donate, deposit, and then try withdrawal.\
Whether it's accepted or rejected, no confirmation transaction is on the Stacks chain.

Here's some logs to show reverse of receiving signer decisions:

### sbtc-signer-2

```bash
{"timestamp":"2025-03-11T07:43:17.219844416Z","level":"INFO","message":"Audit: 🚨 Withdrawal Accepted: true","target":"signer::request_decider","filename":"signer/src/request_decider.rs","line_number":310,"spans":[{"public_key":"031a4d9f4903da97498945a4e01a5023a1d53bc96ad670bfe03adf8a06c52e6380","name":"request-decider"},{"chain_tip":"01e945d974168f3d00df103a914a294af9ff1a2bf151dbbd9e9294b5841dcd51","name":"handle_new_requests"},{"name":"handle_pending_withdrawal_request"}]}
{"timestamp":"2025-03-11T07:43:17.22746311Z","level":"INFO","message":"Audit: 🚨 Preserving message","target":"signer::network::libp2p::event_loop","filename":"signer/src/network/libp2p/event_loop.rs","line_number":414,"spans":[{"name":"p2p"},{"name":"swarm"},{"name":"gossipsub"}]}
{"timestamp":"2025-03-11T07:43:17.230662102Z","level":"INFO","message":"Audit: 🚨 Sent preserved and new messages","target":"signer::network::libp2p::event_loop","filename":"signer/src/network/libp2p/event_loop.rs","line_number":412,"spans":[{"name":"p2p"},{"name":"swarm"},{"name":"gossipsub"}]}
>>> {"timestamp":"2025-03-11T07:43:17.231046035Z","level":"INFO","message":"Audit: 🚨 Withdrawal Decision From: PublicKey(PublicKey(469ea4c224962fb6e3517f3d8e585a9e9b4b723ec4ec65eecc77c08672134952abc06886c0abcce53062ffed93044bcc0d6ded2dfd9ad53a8a96ce9ea422200f)), Decision: false, Tx: StacksTxId(5021b5a07ac159991605553480893252856918d0c387e9c41d72298d99ff4636)","target":"signer::request_decider","filename":"signer/src/request_decider.rs","line_number":482,"spans":[{"public_key":"031a4d9f4903da97498945a4e01a5023a1d53bc96ad670bfe03adf8a06c52e6380","name":"request-decider"},{"name":"handle_signer_message"},{"sender":"035249137286c077ccee65ecc43e724b9b9e5a588e3d7f51e3b62f9624c2a49e46","name":"persist_received_withdraw_decision"}]}
>>> {"timestamp":"2025-03-11T07:43:17.23257764Z","level":"INFO","message":"Audit: 🚨 Withdrawal Decision From: PublicKey(PublicKey(469ea4c224962fb6e3517f3d8e585a9e9b4b723ec4ec65eecc77c08672134952abc06886c0abcce53062ffed93044bcc0d6ded2dfd9ad53a8a96ce9ea422200f)), Decision: true, Tx: StacksTxId(5021b5a07ac159991605553480893252856918d0c387e9c41d72298d99ff4636)","target":"signer::request_decider","filename":"signer/src/request_decider.rs","line_number":482,"spans":[{"public_key":"031a4d9f4903da97498945a4e01a5023a1d53bc96ad670bfe03adf8a06c52e6380","name":"request-decider"},{"name":"handle_signer_message"},{"sender":"035249137286c077ccee65ecc43e724b9b9e5a588e3d7f51e3b62f9624c2a49e46","name":"persist_received_withdraw_decision"}]}
{"timestamp":"2025-03-11T07:43:17.234613455Z","level":"INFO","message":"Audit: 🚨 Withdrawal Decision From: PublicKey(PublicKey(5c13f6f0fe364d09cf9e09180a134381b223e0867e4f7fd9cad4230143117300401dbc447a409b5a653ba5a6fee977e4a7979c939fe9fa2f63c73e9e29d373ed)), Decision: true, Tx: StacksTxId(5021b5a07ac159991605553480893252856918d0c387e9c41d72298d99ff4636)","target":"signer::request_decider","filename":"signer/src/request_decider.rs","line_number":482,"spans":[{"public_key":"031a4d9f4903da97498945a4e01a5023a1d53bc96ad670bfe03adf8a06c52e6380","name":"request-decider"},{"name":"handle_signer_message"},{"sender":"02007311430123d4cad97f4f7e86e023b28143130a18099ecf094d36fef0f6135c","name":"persist_received_withdraw_decision"}]}
```

### sbtc-signer-3

```bash
{"timestamp":"2025-03-11T07:43:17.227441694Z","level":"INFO","message":"Audit: 🚨 Preserving message","target":"signer::network::libp2p::event_loop","filename":"signer/src/network/libp2p/event_loop.rs","line_number":414,"spans":[{"name":"p2p"},{"name":"swarm"},{"name":"gossipsub"}]}
{"timestamp":"2025-03-11T07:43:17.229791896Z","level":"INFO","message":"Audit: 🚨 Sent preserved and new messages","target":"signer::network::libp2p::event_loop","filename":"signer/src/network/libp2p/event_loop.rs","line_number":412,"spans":[{"name":"p2p"},{"name":"swarm"},{"name":"gossipsub"}]}
>>> {"timestamp":"2025-03-11T07:43:17.230833799Z","level":"INFO","message":"Audit: 🚨 Withdrawal Decision From: PublicKey(PublicKey(469ea4c224962fb6e3517f3d8e585a9e9b4b723ec4ec65eecc77c08672134952abc06886c0abcce53062ffed93044bcc0d6ded2dfd9ad53a8a96ce9ea422200f)), Decision: true, Tx: StacksTxId(5021b5a07ac159991605553480893252856918d0c387e9c41d72298d99ff4636)","target":"signer::request_decider","filename":"signer/src/request_decider.rs","line_number":482,"spans":[{"public_key":"02007311430123d4cad97f4f7e86e023b28143130a18099ecf094d36fef0f6135c","name":"request-decider"},{"name":"handle_signer_message"},{"sender":"035249137286c077ccee65ecc43e724b9b9e5a588e3d7f51e3b62f9624c2a49e46","name":"persist_received_withdraw_decision"}]}
>>> {"timestamp":"2025-03-11T07:43:17.23245868Z","level":"INFO","message":"Audit: 🚨 Withdrawal Decision From: PublicKey(PublicKey(469ea4c224962fb6e3517f3d8e585a9e9b4b723ec4ec65eecc77c08672134952abc06886c0abcce53062ffed93044bcc0d6ded2dfd9ad53a8a96ce9ea422200f)), Decision: false, Tx: StacksTxId(5021b5a07ac159991605553480893252856918d0c387e9c41d72298d99ff4636)","target":"signer::request_decider","filename":"signer/src/request_decider.rs","line_number":482,"spans":[{"public_key":"02007311430123d4cad97f4f7e86e023b28143130a18099ecf094d36fef0f6135c","name":"request-decider"},{"name":"handle_signer_message"},{"sender":"035249137286c077ccee65ecc43e724b9b9e5a588e3d7f51e3b62f9624c2a49e46","name":"persist_received_withdraw_decision"}]}
{"timestamp":"2025-03-11T07:43:17.233707799Z","level":"INFO","message":"Audit: 🚨 Withdrawal Decision From: PublicKey(PublicKey(80632ec5068adf3ae0bf70d66ac93bd5a123501ae0a445894997da03499f4d1a3db3dcc5e847954cacef01cbf0b938c98738caa73ba4aaa46b3eaddb65c2940e)), Decision: true, Tx: StacksTxId(5021b5a07ac159991605553480893252856918d0c387e9c41d72298d99ff4636)","target":"signer::request_decider","filename":"signer/src/request_decider.rs","line_number":482,"spans":[{"public_key":"02007311430123d4cad97f4f7e86e023b28143130a18099ecf094d36fef0f6135c","name":"request-decider"},{"name":"handle_signer_message"},{"sender":"031a4d9f4903da97498945a4e01a5023a1d53bc96ad670bfe03adf8a06c52e6380","name":"persist_received_withdraw_decision"}]}
```

In the above log, from the public key `469e...200f`, `sbtc-signer-2` received `false` and `true`, but `sbtc-signer-3` received `true` and `false`.\
As a result, `sbtc-signer-2` has `false` in its database, and `sbtc-signer-3` has `true` in its database.
