# #39910 \[W\&A-Medium] Numerous replay attacks (with arbitrary data) to protected endpoints are possibl

**Submitted on Feb 10th 2025 at 15:10:03 UTC by @anton\_quantish for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39910
* **Report Type:** Websites and Applications
* **Report severity:** Medium
* **Target:** <https://github.com/shardeum/json-rpc-server/tree/itn4>
* **Impacts:**
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.

## Description

## Brief/Intro

An attacker has the opportunity to, being able to intercept signed '/api/subscribe' (or any other protected endpoint) request once, execute it on multiple other JSON-RPC servers with arbitrary request body.

## Vulnerability Details

Some of the API endpoints are protected with the `debugMiddleware`, for instance:

* `/api/subscribe` which changes the main validator node the JSON-RPC server communicates to;
* counters debug endpoints;
* logs debug endpoints.

This protection allows them to be used only either if the JSON-RPC server is in debug mode, or if the request is signed with the developer key.

Below is the corresponding check code:\
<https://github.com/shardeum/json-rpc-server/blob/616fe1007568db801b0433ece9ef822a0e39d5f6/src/middlewares/debugMiddleware.ts#L47-L81>

```js
for (const ownerPk in devPublicKeys) {
  const message = {
    route: _req.route.path,
    count: String(_req.query.sig_counter),
  }
  const sigObj = {
    route: _req.route.path,
    count: String(_req.query.sig_counter),
    requestHash: crypto.hash(Utils.safeStringify(message)),
    sign: { owner: ownerPk, sig: requestSig },
  }
  //reguire a larger counter than before. This prevents replay attacks
  const currentCounter = parseInt(sigObj.count)
  const currentTime = new Date().getTime()
  if (currentCounter > lastCounter && currentCounter <= currentTime + MAX_COUNTER_BUFFER_MILLISECONDS) {
    let verified = verify(sigObj, ownerPk)
    if (verified === true) {
      const authorized = ensureKeySecurity(ownerPk, authLevel)
      if (authorized) {
        lastCounter = currentCounter
        next()
        return
      } else {
        /* prettier-ignore */ nestedCountersInstance.countEvent( 'security', 'Authorization failed for security level: ', authLevel )
        return res.status(403).json({
          status: 403,
          message: 'FORBIDDEN!',
        })
      }
    }
  }
}
```

If the request to a protected endpoint received, and the server is not in the debug mode:

1. The presence of `sig` and `sig_counter` query params is checked
2. It's checked that the request `sig_counter` param **is greater than the `lastCounter` global variable**
3. The **signature of request `route path` and `count`** is checked to be valid for any of the developer authorized public keys.

There are two vulnerabilities:

1. The `lastCounter` is set to 0 when the server starts, which means that the attacker can re-send the signed request to:

* the same server after it has been restarted – the signature will be valid, and the count will be greater than 0;
* **any other JSON-RPC server** which has not received any protected calls yet (it's a default state of any server) or has received them earlier than the intercepted request;

2. The `request data` is not included in the signed object, which means that the attacker can arbitrary alter the request params.

Moreover, the `/api/subscribe` request is HTTP-based (without encryption) and uses the GET method. All the data, including the signature, is contained in the request URL. This means that it could easily be intercepted or even logged by various network devices, such as routers (and then intercepted from there).

## Impact Details

Thus, being able to once intercept the signed request to the `/api/subscribe` endpoint of any JSON-RPC server, the attacker can then arbitrary change the validator node ip and/or port inside the request, and successfully re-send it to other JSON-RPC servers having the same dev public keys configured that, in turn, can paralyze or disrupt the whole network.

The other debug endpoints are also vulnerable so the requests to them could also be forged.

Despite the impact falls under critical, the initial request intercepting is needed, so I downgraded the severity to `High`.

## Proof of Concept

First, let's imitate the authorized `/api/subscribe` request.

With node REPL:

```js
c = require('@shardus/crypto-utils')
c.init('69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc')
utils = require('@shardus/types')
message = {
  route: '/api/subscribe',
  count: '12345',
}
sigObj = {
  route: message.route,
  count: message.count,
  requestHash: c.hash(utils.Utils.safeStringify(message))
}
c.signObj(sigObj, 'YOUR_AUTHORIZED_DEV_PRIVATE_KEY', 'YOUR_AUTHORIZED_DEV_PUBLIC_KEY')
```

You will see the request signature.

Send it with curl:

```
curl 'http://127.0.0.1:8080/api/subscribe?sig_counter=12345&sig=f327b7172b297d918ebd6fded5dbf4f008768f0c431660d4a7f7ebfc575d61c4f81b44d3a9e28c462dbbcbbce2ba7c1cef15d98ed306d08b820df90730f0350c09fed708000e521d9ba8e4974beef5eb0c93ab22fab2399e22d0a3f530dc727c&ip=127.0.0.1&port=11111' -H 'Content-Type: application/json'
```

You should see the request is successfully processed (authorization is passed).

Now, as an attacker, arbitrary change the `ip` and/or `port` variables within the request and send it to any other JSON-RPC server. You will see the request is successfully processed again and the JSON-RPC server's validator node is changed. You can send it to as many servers as you want if they hasn't received any signed requests yet (because it bumps the internal replay-protection counter) or if they has received them earlier than the counter in your intercepted request.

## Mitigation

1. Initialize the `lastCounter` to be equal to the current time on server start, it will prevent the same-server replay attack
2. Include something instance-specific in the request signature (server IP maybe or something similar), it will prevent the cross-server replay attacks
3. It's better to also sign the request data itself, not only the route
