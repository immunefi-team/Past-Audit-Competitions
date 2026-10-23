# #39942 \[W\&A-Medium] Archiver is still vulnerable to replay attack to \`/set-config\`

**Submitted on Feb 11th 2025 at 11:27:50 UTC by @anton\_quantish for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39942
* **Report Type:** Websites and Applications
* **Report severity:** Medium
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.

## Description

## Brief/Intro

I decided to check how you addressed the #35824 report, and it turns out that the vulnerability still exists.

<https://github.com/shardeum/archiver/blob/cbe1d515e91058d17fa483f84361992cd3d1cf9c/src/DebugMode.ts#L5-L48>

According to the current policy (refer the attached screenshot),

> Bugs from previous bounties are in scope unless explicitly said otherwise. Reports 33428, 33655, 33963, 34508, 33576, 34053, 36024, 36025, 36025 are OOS.

Since `35824` is not mentioned in the list above, I believe the issue is in scope.

**Moreover**, I discovered the circumstances which bumps the severity in my opinion.

## Vulnerability Details

With your permission, I will not duplicate all the report details here. Please refer to the previous report on\
<https://bugs.immunefi.com/dashboard/submission/35824>

Briefly, the `debugMiddleware` checks the protected endpoints request authenticity by checking:

* the signature (only the `request path` and the `counter` are signed) validity for the dev key;
* that the counter value is greater than the `lastCounter` global variable.

But the `lastCounter` value is initialized with 0 on the server start, so the replay attack is possible after the server is restarted. The replayed request data could also be arbitrarily modified since the data itself is not signed.

As for the **Extra circumstances** bumping the severity, the intercepted request could not only be re-send to the same archiver after it's restarted, but also to **any other archiver** sharing the same dev keypair which either:

* Hasn't received any signed requests yet (the `lastCounter` is 0 on such archivers). It's the default state of the archive server.
* Has received signed requests earlier than the `counter` in the intercepted request.

## Impact Details

Being able to once intercept the signed request to the `/set-config` endpoint of any archive server, the attacker can then arbitrary change the configuration inside the request, and successfully re-send it to other archive servers having the same dev public keys configured that, in turn, can potentially paralyze or disrupt the whole network.

The other debug endpoints are also vulnerable so the requests to them could also be forged.

Despite the impact falls under critical, the initial request intercepting is needed, so I downgraded the severity to High.

## Proof of Concept

1. Generate the valid counter and signature with node REPL shell:

```js
const crypto = require('@shardus/crypto-utils')
crypto.init(YOUR_ARCHIVER_HASH_KEY)
obj = {route: '/set-config', count: (new Date().getTime()).toString()}
crypto.signObj(obj, YOUR_ARCHIVER_SECRET_KEY, YOUR_ARCHIVER_PUBLIC_KEY)
obj
```

the result is something like

```js
{
  route: '/set-config',
  count: '1728500989222',
  sign: {
    owner: '31ba246ea6baef8f86a8b6cb2b7c84b0223c5975f8c6974d74d856efe94728e1',
    sig: '01cdd7d7cee5b076b9716f66b3d09ebc726fec2c5bfd74437751e43d1c6f4ebef650097d6ad9f718fc7fb792f7697d8051f36d5abc0048c3bf21c30c6d972e03eb3bd6973dd7f359ea8825bdfaed6bdee66f95a906c603028c80c5fa9db006f6'
  }
}
```

2. Send the test (legitimate) request with CURL:

```bash
curl 'http://127.0.0.1:4000/set-config?sig_counter=1728500989222&sig=01cdd7d7cee5b076b9716f66b3d09ebc726fec2c5bfd74437751e43d1c6f4ebef650097d6ad9f718fc7fb792f7697d8051f36d5abc0048c3bf21c30c6d972e03eb3bd6973dd7f359ea8825bdfaed6bdee66f95a906c603028c80c5fa9db006f6' -XPATCH -H 'Content-Type: application/json' -d '{"VERBOSE": false}'
```

Make sure the response contains `"VERBOSE": false`

3. Send it again and make sure it's forbidden because of invalid counter.
4. Alter the request data arbitrarily, for instance to `{"VERBOSE": true}`
5. Either:

* re-send the altered request to any other archiver sharing the same dev keypair;
* restart the archiver and re-send the altered request there.

Make sure the request is successfully processed and the response contains `"VERBOSE": true`.
