# #39979 \[BC-Critical] Total network shutdown via fixDeserializedWrappedEVMAccount call through binary\_repair\_oos\_accounts endpoint

**Submitted on Feb 12th 2025 at 03:05:39 UTC by @neploxaudit for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39979
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds
  * Network not being able to confirm new transactions (total network shutdown)
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

This report describes a vulnerability in the same mechanism as our report #33925 (<https://bugs.immunefi.com/dashboard/submission/33925>), which we submitted during the first Shardeum Core boost.\
Even though the internal plaintext endpoint `repair_oos_accounts` is removed from the Shardus Core component (<https://github.com/shardeum/core/tree/dev),\\>
and the internal binary endpoint `binary_repair_oos_accounts` has received multiple improvements to protect it from the reported\
consensus issues (input validation, `proposalHash` validation, `txReceipt` validation using `robustQuery`, `verifyAppliedReceipt`),\
the `fixDeserializedWrappedEVMAccount` sink function can still be exploited to DOS Shardeum nodes with minimal requirements for the attacker.

## Vulnerability Details

Our report #33925 utilized the internal endpoint `repair_oos_accounts` (since then removed in commit <https://github.com/shardeum/core/commit/7d8877b7e1a5b18140f898a64b932182d8a35298)\\>
to pass arbitrary objects to `fixDeserializedWrappedEVMAccount` through the `calculateAccountHash` method.\
However, since then these methods have not been fixed: `fixDeserializedWrappedEVMAccount` (<https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/shardeum/wrappedEVMAccountFunctions.ts#L86>) still accepts\
objects without validation and will pass their fields `storageRoot`, `codeHash`, `codeByte`, `value` to the `Uint8Array.from` constructor,\
which expects "Array-like" objects with the "length" property set.`calculateAccountHash` also hasn't changed, which can be verified using `git blame` view: <https://github.com/shardeum/shardeum/blame/167e48478403918468410dd7562929653d5b9f6b/src/index.ts#L6503>.

The `calculateAccountHash` "middleman" function is still called throughout Shardeum repos,\
however the most realistically exploitable source is in the `binary_repair_oos_accounts` endpoint: <https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/state-manager/AccountPatcher.ts#L665.\\>
If we check the implementation of `binary_repair_oos_accounts`, we can see that the first usage of the `accountData.data` object\
is exactly where we need it to be for the exploit - in `calculateAccountHash`.\
The only other place in `binary_repair_oos_accounts`'s execution flow where the `accountData.data` object is manipulated\
is the deserializer function, `deserializeRepairOOSAccountsReq` (<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/types/RepairOOSAccountsReq.ts#L49>), which calls `deserializeWrappedData` (deserializeWrappedData) for the `appData` field,\
which in turn just calls `app.binaryDeserializeObject` (implemented in the Shardeum node, <https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/index.ts#L7871>).

`app.binaryDeserializeObject` will deserialize arbitrary objects through the `accountDeserializer` (<https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/types/Helpers.ts#L88)\\>
when we serialize a payload with an unknown type: <https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/types/Helpers.ts#L112>. The deserialized object will be placed in `appData`'s `data` field,\
and will not be validated using AJV because the schema marks `schemaWrappedData.data` as an "opaque data blob" (<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/types/ajv/WrappedData.ts#L8>).

Since `appData.data` will not be checked in any way before it is passed to `calculateAccountHash`,\
an attacker can craft an otherwise completely valid `binary_repair_oos_accounts` call (with valid txReceipt, signatures, etc),\
but replace the `appData.data` payload with a malicious object such as this one:

```js
{"accountType": 0, "account": {"storageRoot": {"data": {"length": 4294967294}}}}
```

To cause the target node to start consuming CPU and Memory, eventually making it come to a hault (we have shown this in our original report #33925).

## Impact Details

Please check the Impact Details in our report #33925, as they still hold for this report.

Because `binary_repair_oos_accounts` requires no more than a valid transaction from the Shardeum blockchain\
to be called, it can be exploited by any validator node which is part of the network (because the endpoint is internal).\
This malicious node can disable all other nodes one by one and takeover the network or just bring it down completely.

## References

* Our (Neplox) report #33925: <https://bugs.immunefi.com/dashboard/submission/33925>
* Report #33925 archived on Immunefi's GitHub: <https://github.com/immunefi-team/Bounty\\_Boosts/blob/main/Shardeum%20Core/33925%20-%20%5BBC%20-%20Critical%5D%20Improper%20input%20validation%20in%20fixDeserializedWra....md>
* Current `fixDeserializedWrappedEVMAccount` implementation in `dev` branch: <https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/shardeum/wrappedEVMAccountFunctions.ts#L86>

## Link to Proof of Concept

<https://gist.github.com/renbou/8ae48a8b3c9bb1f3c106a2aa1ec839a2>

## Proof of Concept

## Proof of Concept

To avoid writing up the same PoC as in report #33925, we will simply describe the\
modifications that are need in order to make the PoC setup work now.\
The new PoC also implements also the DoS exploit, but uses the `binary_repair_oos_accounts` endpoint instead of the old one.

The attached gist contains [dos-poc.ts](https://gist.githubusercontent.com/renbou/8ae48a8b3c9bb1f3c106a2aa1ec839a2/raw/dc7f9794e5a1b3c3abea7438f8fcbc3269d20f81/dos-poc.ts), [config.ts](https://gist.githubusercontent.com/renbou/8ae48a8b3c9bb1f3c106a2aa1ec839a2/raw/dc7f9794e5a1b3c3abea7438f8fcbc3269d20f81/config.ts), [package.json](https://gist.githubusercontent.com/renbou/8ae48a8b3c9bb1f3c106a2aa1ec839a2/raw/dc7f9794e5a1b3c3abea7438f8fcbc3269d20f81/package.json) and [tsconfig.json](https://gist.githubusercontent.com/renbou/8ae48a8b3c9bb1f3c106a2aa1ec839a2/raw/dc7f9794e5a1b3c3abea7438f8fcbc3269d20f81/tsconfig.json) files,\
which is a bit more complex than before, however allows using Shardeum's methods instead of\
implementing our own.

Do note that we had trouble running the PoC with different node setups and thus opted\
for a 10-node setup, which worked the most stable for now, considering the ongoing issues\
with transaction handling in Shardeum. With 32 node setups (and even with 10)\
we received `stuck_in_consensus_3` errors on the `dev` branch of Shardeum,\
which we have reported in the competition Discord channel.\
As such, the PoC might be unstable due to Shardeum itself being unstable.\
Overall, however, it does not depend on the number of nodes in the network,\
and can be used to take down a network of any size (possibly with some changes).

**WARNING!** the exploit causes each validator node to slowly accumulate up to a 4 gigabytes (the 4294967294 constant in the exploit) worth of memory, so make sure to run it in an unimportant environment and be ready for possible crashes and OOM errors. The value 4294967294 can also be decreased for running the setup and exploit locally, it was chosen in order to clearly demonstrate the impact of node DOS via memory and CPU resource consumption.

## Local Shardeum network setup

The setup is pretty much the same as before, however now Shardeum uses\
NodeJS version v18.19.1, so the following `asdf` (version 0.16) commands can be used to install it:

```bash
asdf install nodejs v18.19.1
asdf set nodejs v18.19.1
```

Additionally, the `network-10.patch` file (<https://gist.githubusercontent.com/renbou/8ae48a8b3c9bb1f3c106a2aa1ec839a2/raw/dc7f9794e5a1b3c3abea7438f8fcbc3269d20f81/network-10.patch>) now includes a change to modify\
the new `flexibleRotationDelta` setting to `0`, as suggested by `mgthura` from the Shardeum team in the Discord server. Without this change, nodes immediately start rotating once the network reaches `processing` mode, which makes testing more difficult.\
The patch also sets up only 10 nodes instead of 32.

**NOTE:** Make sure that the `network-10.patch` is applied properly, as it also configures `genesis.json`\
to have an additional account for the attacker, which is used to create a simple transfer transaction in the PoC.

To use the Shardeum node with the latest `core` version from the `dev` branch,\
we used `npm link` as specified in the `core` README: <https://github.com/shardeum/core/blob/dev/README.md>.

## JSON-RPC API setup

The setup of the JSON-RPC API only differs in the NodeJS version as well.\
The JSON-RPC API is needed in this PoC to simplify sending transactions,\
since the `binary_repair_oos_accounts` requires a valid transaction receipt to be passed to it.

## Proof of Concept exploit

The overall configuration and setup of the PoC is the same as in our report #33925,\
however it uses the updated Shardeum libraries (their versions are specified in the `package.json` file).\
To run the PoC, place the `dos-poc.ts`, `config.ts`, `package.json` and `tsconfig.json` files in the same directory, run `npm install`, compile the PoC using `npx tsc`, and then run it using `node ./build/dos-poc.js`.

**NOTE:** Make sure to set `ATTACKER_NODE_SECRETS` in the `dos-poc.ts` file to the value of `secrets.json` from one of the nodes in the network to simulate a malicious node.

Here are the main changes to the PoC, which come from the need of\
using the `binary_repair_oos_accounts` endpoint instead of the plaintext one:

* The `init` method configures a lightweight version of the Shardus core requirements\
  to be able to send requests over the internal network protocol;
* `askBinary` from `p2p/Comms.ts` is used for requesting the valid signed receipt\
  of the transaction via `binary_request_receipt_for_tx`, and for sending the\
  exploit request via `binary_repair_oos_accounts`.

Since no new validation mechanisms have been implemented for `appData.data`,\
we just use the same `appData.data` payload as before (mentioned in the description of the report).

After running the PoC, you can check that any requests to the nodes hang\
(e.g. `http://localhost:{port}/nodeinfo`), and that the network monitor on `http://localhost:3000` soon starts showing all the nodes (except the malicious one) as dead.\
An example of the terminal output of the PoC and later `curl` requests to the\
nodes is attached to the gist: <https://gist.githubusercontent.com/renbou/8ae48a8b3c9bb1f3c106a2aa1ec839a2/raw/dc7f9794e5a1b3c3abea7438f8fcbc3269d20f81/out.log.\\>
The `curl` requests at the end show that all nodes except the malicious one stop responding.

A screenshot showcasing the increased CPU and memory usage of the nodes after running\
the PoC is also attached to the report.
