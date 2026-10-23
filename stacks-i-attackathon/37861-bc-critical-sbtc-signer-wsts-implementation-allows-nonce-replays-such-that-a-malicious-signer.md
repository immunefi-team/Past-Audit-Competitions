# #37861 \[BC-Critical] SBTC Signer WSTS implementation allows nonce replays such that a malicious sign

**Submitted on Dec 17th 2024 at 19:24:33 UTC by @throwing5tone7 for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37861
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/Trust-Machines/wsts>
* **Impacts:**
  * Direct loss of funds
  * Network not being able to confirm new transactions (total network shutdown)
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

The SBTC Signers collaborate to achieve threshold signing of Stacks & Bitcoin transactions using the WSTS fire protocol designed by Trust Machines. The implementation used is in the WSTS repo owned by Trust machines. This implementation allows nonces used during signing operations to be repeated, because it does not clear out nonces after use. By abusing the protocol a malicious signer who is the coordinator of a signing round can request & receive multiple signatures from the other signers using different messages but the same nonce values. Once they have this they can derive the secrets of all other signers, and take over control of the group signer address - this allows them to completely take over the signing process since they could rotate to a single signer key of their own. At this point, they can transfer all of the BTC out of the pegged UTXO, and mint / burn etc sBTC as much as they like, and completely stop the protocol.

### Scope considerations

I am aware that the WSTS repo isn't strictly in scope for this Attackathon. However, I confirmed on public messages with the Stacks team that "a bug in WSTS which has a direct impact on the sBTC deposit flow" would be considered in scope. See the message from the technical team member here - <https://discord.com/channels/787092485969150012/1311889396542214276/1314577660017643580>. Since the WSTS flaw can ultimately lead to total loss of funds and hard freezing of the protocol I believe it is in scope. My PoC is a demonstration of a malicious signer deriving the secrets running in the context of sBTC signers coordinating to process deposit - so I clearly demonstrate that this flaw does affect the sBTC deposit process and isn't just a lower level bug that doesn't apply to sBTC's use of WSTS.

## Vulnerability Details

During sBTC signature signing, the WSTS signer process ultimately calls through to v2::Party::sign\_with\_tweak. As you can see from the code, this function relies on a nonce value that has been pre-generated (`self.nonce`), and the function itself does not clear or regenerate the nonce after use. This is a critical security vulnerability as any time attacker can observe multiple signatures on the same nonce values but over different messages, they can derive the underlying secret shares that authenticate the parties in the threshold signature protocol.

Taken from <https://github.com/Trust-Machines/wsts/blob/ebd7d7775ad5e44cdbf4f5c1fb468bdf6c467265/src/v2.rs> - lines 193..244 (this is the revision that is used by sBTC currently but the code for this function is identical on the main branch too).

```
    pub fn sign_with_tweak(
        &self,
        msg: &[u8],
        party_ids: &[u32],
        key_ids: &[u32],
        nonces: &[PublicNonce],
        tweak: Option<Scalar>,
    ) -> SignatureShare {
        // When using BIP-340 32-byte public keys, we have to invert the private key if the
        // public key is odd.  But if we're also using BIP-341 tweaked keys, we have to do
        // the same thing if the tweaked public key is odd.  In that case, only invert the
        // public key if exactly one of the internal or tweaked public keys is odd
        let mut cx_sign = Scalar::one();
        let tweaked_public_key = if let Some(t) = tweak {
            if t != Scalar::zero() {
                let key = compute::tweaked_public_key_from_tweak(&self.group_key, t);
                if key.has_even_y() ^ self.group_key.has_even_y() {
                    cx_sign = -cx_sign;
                }

                key
            } else {
                if !self.group_key.has_even_y() {
                    cx_sign = -cx_sign;
                }
                self.group_key
            }
        } else {
            self.group_key
        };
        let (_, R) = compute::intermediate(msg, party_ids, nonces);
        let c = compute::challenge(&tweaked_public_key, &R, msg);
        let mut r = &self.nonce.d + &self.nonce.e * compute::binding(&self.id(), nonces, msg);
        if tweak.is_some() && !R.has_even_y() {
            r = -r;
        }

        let mut cx = Scalar::zero();
        for key_id in self.key_ids.iter() {
            cx += c * &self.private_keys[key_id] * compute::lambda(*key_id, key_ids);
        }

        cx = cx_sign * cx;

        let z = r + cx;

        SignatureShare {
            id: self.party_id,
            z_i: z,
            key_ids: self.key_ids.clone(),
        }
    }
```

In normal operation, the protocol does not request multiple messages to be signed with the same nonce (a bona fide coordinator requests a nonce commitment immediately before requesting each signature share), so in normal operation there is no chance to exploit the repeated nonce weakness. Thus a malicious signer must maliciously modify the protocol message ordering in order to be able to observes signatures sharing the same nonces and reveal the secrets.

The threat model is an SBTC signer who modifies the code that their instance of their signer process runs. The steps that a malicious actor is required to implement in their signer code base are as follows:

1. Signer waits until they are a coordinator for a round with at least 3 messages to sign (in my PoC these are 3 bitcoin signature messages caused by 2 sBTC deposit requests in a BTC block)
2. Rather than launch `n` signature rounds to sign `n` messages as the normal protocol does, launch a single round that signs `n` messages sequentially:
   * The protocol runs as normal for nonce generation - the coordinator just requests the signers to generate a nonce wrt to the first message
   * The coordinator then loops around the "request signature shares" and "gather shares" steps one message at a time, only exiting the "gather shares" stage when all `n` messages are signed
3. The malicious coordinator derives all the other participants secret shares - at this point they can completely take over the process provided they act before any further Distributed Key Gen (DKG) rounds that would change the shares

It is pretty straightforward to modify the sBTC signer code to do this, as demonstrated from my PoC patch.

### Deriving secrets

Note that at the point of exiting the "gather shares" process for all messages, with at least 3 messages, the coordinator has observed 3 signatures per participant.

Considering only the first 3 signatures from a given participant they know:

* `z1 = r_adjust1 * (nonce.d + nonce.e * compute::binding(signer_id, public_nonces, msg1)) + priv_key * Lambda * c1 * cx_sign`
* `z2 = r_adjust2 * (nonce.d + nonce.e * compute::binding(signer_id, public_nonces, msg2)) + priv_key * Lambda * c2 * cx_sign`
* `z3 = r_adjust3 * (nonce.d + nonce.e * compute::binding(signer_id, public_nonces, msg3)) + priv_key * Lambda * c3 * cx_sign`

NOTE: I've simplified the above slightly from the WSTS code to account for the fact that each participant only has a single private key share in sBTC (they all have the same "weight" in the protocol), and to define some terms to make the explanation clearer. I defined r\_adjust for a message as either -1 or +1 depending on whether the condition `tweak.is_some() && !R.has_even_y()` from the code is met or not, just making the maths a bit more convenient. Lambda is the publicly calculable contribution of the participant's secret share to the overall secret (the interpolation coefficients).

The attacker now knows or can calculate all of the following with respect to this participant:

* All the signatures z1, z2, z3
* All of the r\_adjust values which only derive from publicly known data
* All of the messages msg1, msg2, msg3
* The signer ID & public nonces in the scheme
* Hence all of the results of `compute::binding` that are relevant
* cx\_sign and Lambda which are derivable from publicly known data

This means they have 3 equations with 3 unknown variables (`nonce.d`, `nonce.e` and `priv_key`) and can solve simultaneous equations to find the value of `priv_key` which is the sole secret share of the participant (for more details refer to `bad_fire_coordinator::derive_secrets` in the patch I provided).

They can also repeat this derivation process for the other participants who provided multiple signatures to reveal enough secret shares to control the signing process.

## Impact Details

As described above, any **single** malicious signer engaging in the sBTC deposit processing flow can reveal the current secret shares of all other participants. If they recover at least `t - 1` such secret shares (where `t` is the threshold of the signature scheme) they can act unilaterally on behalf of the group. At this point they would be most likely to immediately:

* Change the group key on both the sBTC and BTC sides of the protocol to be a singular key that they know and is secret from all other participants
* Drain the BTC wallet of all pegged funds
* Mint as much sBTC as they like

The ongoing impacts of this are clearly:

* Direct loss of funds
* Freezing of the protocol - they now control the important addresses that govern the protocol so can refuse to process any further deposits or just process them in a malicious way that is detrimental to the users and protocol

## Link to Proof of Concept

<https://gist.github.com/throwin5tone7/a8a3de37b0e713c09d68e222f985e06e>

## Proof of Concept

## Proof of concept

### High level overview

As I stated in the bug description, the threat model is a malicious sBTC signer who modifies the code that their signer process runs to exploit the flaw. The PoC for this is a modified sBTC code-base where the signers can either act as normal or be malicious (this can be controlled by a command line argument or environment variable). For the legitimate signers you will see that the only meaningful code change is to log out their secret shares to demo the exploit clearly (these logs are not observed by or communicated to the malicious signer).

The malicious signer is a normal signer with a slightly modified transaction coordinator (which is a copy-paste and then modified version of the legitimate version, just to keep the malicious & non-malicious code paths as cleanly separate as possible). The malicious signer works with a malicious WSTS state machine which drives a modified version `bad_fire_coordinator` of the `fire::coordinator` implementations from the WSTS repo. The malicious signer abuses the protocol to request multiple signatures over differing messages in a single protocol run, as described in the bug description.

In order to be able to derive the secrets, the malicious coordinator needs to be able to see at least 3 signatures. Since I believed that the signers would reject messages they weren't expecting to sign (NOTE: I did not actually verify that they would reject them), I designed the PoC such that the signer will execute the attack on legitimate bitcoin deposit processing, when there are at least 2 deposits to process (which requires 3 signatures). In order to be able to see a block with enough deposits when the malicious signer is the coordinator, I run a script that feeds deposits repeatedly every few seconds until we see a BTC block where the bad signer is the coordinator. NOTE that if sBTC is even remotely successful the likelihood of at least 2 sBTC deposit requests in a single BTC block would be very high, and the malicious actor can just wait until it happens.

Thus the components of the PoC are:

* Two sBTC signers running normally (docker containers named `sbtc-signer-1` and `sbtc-signer-2`)
* One sBTC signer (running as docker container named `sbtc-signer-3`) that uses malicious code for the transaction coordination function (especially wrt to coordinating WSTS)
* A script to repeatedly send deposits to trigger the condition where the bug can be exploited (I just extend the `signers.sh` script to allow this)
* All of the other normal running components (bitcoin, mining, stacks etc), acting normally, as achieved by launching with `make devenv-up`

### Detailed reproduction instructions

* In order to set up the components as described above - apply my `sbtc-repo.patch` from the GIST linked to the SBTC repo directly on top of commit `f07f68b73db13e80c16fa058ba806fb146090862` (head of the `immunefi_attackaton_0.9` branch as of 17th Dec 17:00 UTC).
  * NOTE that this branch of SBTC uses a specific commit of WSTS, but I have also checked the bug can be reproduced with a fully up to date version of WSTS (by linking to a local download of it in my testing)
* You can use `git diff` to check this patch versus the base commit and you will see that:
  * `docker/docker-compose.yml` has been changed to launch SBTC signer 3 with an additional command line arg to make it act maliciously
  * Several files have been added to represent the "malicious code path":
    * `signer/src/bad_transaction_coordinator.rs` which is a lightly modified version of `signer/src/transaction_coordinator.rs` that uses
    * `signer/src/bad_wsts_state_machine.rs` which is a lightly modified version of `signer/src/bad_wsts_state_machine.rs` that uses
    * `signer/src/bad_fire_coordinator.rs` which is a more modified version of `src/state_machine/coordinator/fire.rs` from the WSTS repo - the secret derivation and most of the protocol abuse happens here
    * `signer/src/lib.rs` has been updated to add these modules
  * `signer/src/main.rs` has been modified to invoke `signer/src/bad_transaction_coordinator.rs` if a command line argument or env var specifies that the current process is a "bad guy"
  * `signer/src/wsts_state_machine.rs` has been modified only to log out the secret share of the SBTC signer to be able to show the attack has worked more easily
  * `signers.sh` has an extra command `repeat-deposit` that does an initial donation and then runs a deposit request every 5s until cancelled
* Ensure you build the SBTC signer components to reflect the patched changes e.g. `docker compose -f docker/docker-compose.yml --profile sbtc-signer build`
* Launch the components with `make devenv-up`
* Wait for the sBTC deposit flow to be ready (i.e. wait until you see the `rotate-keys-wrapper` transaction as mentioned in your README)
* Check the docker logs to see the secrets for signers 1 & 2:
  * run `docker logs sbtc-signer-1 | grep IMPORTANT` you should see a line that ends like `### IMPORTANT ### - my secret share is 67SEHrgNq6116Wuex3gdap9icQZtzbv5XSd5aQbihXYR`
  * repeat for signer 2 `docker logs sbtc-signer-2 | grep IMPORTANT`
* Then tail the logs for the malicious signer, watching for when it will output the secrets - `docker logs -f sbtc-signer-3 | grep 'DERIVED SECRET'`
* Now in a separate shell, run the script to repeat deposits - `./signers.sh repeat-deposit`
  * As soon as a BTC block occurs with at least 2 deposit transactions that the malicious signer is the coordinator for, the attack will run and output the secrets
  * By watching the the malicious signer log output you should see the relevant secrets output after a short window of time
    * You can easily verify by eye that the secrets derived by the malicious signer match the secrets logged out by signers 1 & 2
    * Note that the logs do not easily show which signer has which ID, but the attacker doesn't care if they reveal enough secret shares

### PoC limitations that would be added for an effective attack

Ideally in order to make the PoC even more compelling I would demonstrate transferring the BTC pegged etc, but given the severity of this bug I thought it would be better to just submit ASAP, as it would require a bit more time to develop the code. Hopefully the project team will understand that the secret key shares are the only thing authenticating the sBTC signer group on the BTC and Stacks chains without such a demo, but let me know if required.

In general to turn this into a more effective attack, the attacker would modify the code further to:

* Avoid signing for their own key when doing a malicious protocol run
* Immediately carry out the nefarious impacts (transfer wallets to their own address, drain the peg, mint etc) before any further DKG can run

This is a bit more coding work, but is totally feasible.
