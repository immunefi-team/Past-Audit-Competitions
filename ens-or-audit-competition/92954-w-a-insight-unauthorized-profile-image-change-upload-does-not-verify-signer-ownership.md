# 92954 w a insight unauthorized profile image change upload does not verify signer ownership

**Submitted on Sep 13th 2026 at 13:07:52 UTC by @bugtester for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92954
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Summary

The user profile-image upload worker does not verify that the signer is authorized to manage the ENS name being uploaded to.

The signed message contains the target ENS name, but the recovered signer is never checked against the ENS name's owner or authorized manager.

This allows any wallet to upload an image to another user's existing avatar/header slot without authorization.

ecause the upload request body carries **no `name` field** (`{expiry, dataURL, sig, unverifiedAddress}`, `profileImageUpload.ts:156-161`), the worker's only source for the name is the URL path. Any wallet can therefore:

1. compute `hash = sha256(jpegBytes)`,
2. sign `Upload{upload, expiry, name: <VICTIM>, hash}` with **its own** Any wallet (attacker ) key (the signature is valid — it's over the victim name, which matches the path),

Check Attached Screenshot for proof

## Impact

An attacker can replace the avatar or header of an ENS name that already uses the upload feature, without any interaction or permission from the victim.

This results in an unauthorized profile modification

## Remediation

Verify that the recovered signer owns or is authorized to manage the ENS name before accepting the upload.

## Link to Proof of Concept

<https://gist.github.com/piyushshukla599/a1afdf5323d5a6313f7ef4050b69c2bc>

## Proof of Concept

## PoC

A PoC is included with the submission:

`apps/manager/_poc_avatar_ownership_check.mjs`

It generates a fresh random wallet, signs an upload request containing the victim's ENS name, uploads an attacker-controlled JPEG, verifies that both the storage and public render changed

Run from `apps/manager`:

```bash
SEND=1 VICTIM_NAME='name.eth' IMG_FILE=./attacker.jpg \
  node _poc_avatar_ownership_check.mjs
```

The PoC also supports a dry run.

I reproduced the issue on Sepolia using a fresh wallet that did not own the test ENS name. The upload succeeded, and the public renderer returned the attacker's image. The original image was then restored.

```
attacker signer   : 0xfEa5155950026074D6aAbd7e3925451f493Da56a
victim name       : "enss.eth" (NOT owned by signer)
signed name       : "enss.eth" (== path, so digest matches)
upload endpoint   : https://avatar-upload-staging.ens-cf.workers.dev/sepolia/enss.eth
image             : pwn.jpg (10277 bytes)
expected hash     : 735d71e86bb75f40713ae68bbeb63c0fff615d116a77940771696d5c8d29af9f
raw-store readback: https://avatar-upload-staging.ens-cf.workers.dev/sepolia/enss.eth?v=1789301726456
public render     : https://ens-metadata-v2.ensdomains.workers.dev/sepolia/avatar/enss.eth

-- firing PUT --
PUT status        : 200 {"message":"uploaded"}
```

## Relevant Code

`apps/manager/src/features/profile/service/profileImageUpload.ts`

The upload body contains:

```
{ expiry, dataURL, sig, unverifiedAddress }
```

It does not contain the ENS name. The worker therefore uses the name from the URL path, but fails to verify that the recovered signer is authorized to manage that name.
