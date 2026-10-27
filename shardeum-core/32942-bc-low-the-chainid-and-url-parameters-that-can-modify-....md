# 32942 - \[BC - Low] The ChainID and URL parameters that can modify ...

Submitted on Jul 8th 2024 at 15:54:16 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #32942

Report type: Blockchain/DLT

Report severity: Low

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Shardus server provides two modes; **Debug**, which omits signature checks in sensitive endpoints for an easier developer experience, and **Release** which enforces them for security reasons.

In this report, we'll demonstrate how the authentication check on `shardus-core/src/network/debugMiddleware.ts` can be bypassed, allowing several behaviors, for example: unauthorized modifications to the server configuration of a Shardus Validator making it unable to confirm new transactions.

## Vulnerability Details

As a reference, below is the code of the flagged function that checks if requests to sensitive endpoints are authorized when running in `release` mode - the original version is extensive so only the parts relevant to this report were kept.

```javascript
// This function is used to check if the request is authorized to access the debug endpoint
function handleDebugAuth(_req, res, next, authLevel) {

  try {

    //auth with a signature
    if (_req.query.sig != null && _req.query.sig_counter != null) {

      const devPublicKeys = getDevPublicKeys() // This should return list of public keys

      const requestSig = _req.query.sig

      // Check if signature is valid for any of the public keys
      for (const ownerPk in devPublicKeys) {

        let sigObj = {
          route: _req.route.path,
          count: String(_req.query.sig_counter),
          sign: { owner: ownerPk, sig: requestSig },
        }

        // ...........

        let verified = Context.crypto.verify(sigObj, ownerPk)

        if (verified === true) {

          const authorized = ensureKeySecurity(ownerPk, authLevel)

          if (authorized) {

            lastCounter = currentCounter
            next()
            return

          } else {
  // ...........
}
```

> Snippet of code from: <https://github.com/shardeum/shardus-core/blob/dev/src/network/debugMiddleware.ts#L14-L71>

The address with the correct authorization level must sign an object with the route of the request and a counter that prevents replay attacks.

```
let sigObj = {
  route: _req.route.path,
  count: String(_req.query.sig_counter),
}
```

The counter is provided as a param in the URL (`& sig_counter=`) and the route is taken from the value of the `_req.route.path` variable.

### The problem: Signing only the path to the endpoint, instead of the full URL including the parameters

Let's use the URL that modifies the configuration of a running validator to demonstrate the bug.

In this URL: *<http://192.168.0.15:9001/debug-set-shardeum-flag?key=debugTraceLogs\\&value=false\\&sig\\_counter=xxxx\\&sig=yyyyy>*

The `_req.route.path` is: */debug-set-shardeum-flag*

The authorized address intends to set the value of **debugTraceLogs** to **false**, and sends the request with a signature of the route path (`_req.route.path`) and a `sig_counter`.

Unfortunately, because the `key` and `value` parameters of the URL were not signed a malicious actor can use the same signature and sig\_counter to change any shardeum flag in the server, like the `ChainID` which would render the server unable to process new transactions due to a ChainID mismatch.

The reason is that this malicious URL *<http://192.168.0.15:9001/debug-set-shardeum-flag?key=ChainID\\&value=9999\\&sig\\_counter=xxxx\\&sig=yyyyy>* which includes the same `_req.route.path`, `sig_counter` and `sig`, passes all checks even though it modifies a value that was not intended to be touched.

> Up to this point in the report, for the reader will seem like the "*replay attack check*" will prevent the same `sig_counter` value from being used twice, forcing the malicious actor to intercept, modify, and send his malicious HTTP request before the legit one is processed.
>
> In the next section we'll talk about one of the ways to deal with the replay attack check, making attacks more feasible.

### Reusing signatures from "testnet"/"devnets" in "mainnet"

The ChainID should be required to be signed in every request to a sensitive Validator endpoint, assuring that signatures from testnets and development setups running the "Release" mode for testing purposes can't be re-used in mainnet.

Currently, the only requirement for the signature to be valid is to be signed with the route path and a sig\_counter bigger than the previous one, therefore a request to a non-mainnet Validator in "Release" mode can have its parameters **modified** to cause unauthorized behaviors, and be used in mainnet against the same Validator.

## Impact Details

There are many potential impacts, depending on the type of signature the attacker malleates, for example, using a signature to the path `/debug-set-shardeum-flag` will render a validator unable to process new transactions.

## Proof of Concept

As a proof of concept, create a Node JS project and use express to log the value of the `req.route.path`. The output of the console shows it only includes the exact defined route, but no query params.

> We don't believe that using `req.route.path` in the Shardus Core is a problem by itself, the problem is to not check any of the params sent within the transaction when calculating the signature, which allows for unintended results.

```
const router = (require('express')).Router();

router.get('/some/:long/:path', (req, res) => {
    console.log(req.route.path);      // exact defined route

    res.sendStatus(200);
});

module.exports = router;

```
