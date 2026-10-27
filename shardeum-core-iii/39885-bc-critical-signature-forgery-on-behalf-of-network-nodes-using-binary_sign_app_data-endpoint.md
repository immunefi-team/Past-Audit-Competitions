# #39885 \[BC-Critical] Signature forgery on behalf of network nodes using binary\_sign\_app\_data endpoint

**Submitted on Feb 9th 2025 at 19:55:21 UTC by @neploxaudit for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39885
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Direct loss of funds

## Description

## Brief/Intro

This report describes a bypass to the fix implemented after our report #33632 made in the first Shardeum Core boost.

Despite the `sign-app-data` endpoint being removed from the Shardus Core component (<https://github.com/shardeum/core/tree/dev),\\>
the main `signAppData` method is still exposed via the `binary_sign_app_data` internal endpoint without appropriate validation.\
This allows any validator node part of the currently active nodes to receive near-arbitrary data signed on behalf of other nodes,\
which can be used to gain control of all the consensus protocols implemented in the Shardeum network (transactions, gossip, etc).

## Vulnerability Details

Report #33632 utilized the `sign-app-data` internal endpoint, which has since been disabled (commented out) in commit <https://github.com/shardeum/core/commit/7d8877b7e1a5b18140f898a64b932182d8a35298.\\>
However, the main culprit of the vulnerability, Shardeum's `signAppData` method (<https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/index.ts#L6573>) is still exposed via the binary alternative of the internal endpoint,`binary_sign_app_data` (<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/shardus/index.ts#L3051>).

The `binary_sign_app_data` method applies only basic validation to the `appData` parameter\
in `deserializeSignAppDataReq` (<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/types/SignAppDataReq.ts#L37)\\>
by using `binaryDeserializeObject` to make sure that the `appData` is an object,\
and then by using `verifyPayload(AJVSchemaEnum.SignAppDataReq, result)`,\
which also does not contain proper validation for the `appData` parameter (the AJV schema specifies that `appData` is of unknown contents: <https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/types/ajv/SignAppDataReq.ts#L10>).

The `signAppData` has not been updated since the original report (you can verify it using `git blame` no the `dev` branch <https://github.com/shardeum/shardeum/blame/167e48478403918468410dd7562929653d5b9f6b/src/index.ts#L6573).\\>
As such, it still allows the `originalAppData` parameter, which will be signed,\
to contain arbitrary fields beside the necessary `nominator`, `nominee`, `stake`, `certExp` ones. As we reported in #33632, and @ZhouWu in #34484, this can be utilized\
to get other nodes to sign objects which can then be used maliciously.\
This is the payload we used to exploit the vulnerability in #33632 (and it can still be used now):

```js
{
  nominator: wallet.address,
  nominee: ATTACKER_NODE_SECRETS.publicKey,
  stake: stakeLock,
  certExp: Date.now() + 24 * 60 * 60 * 1000,

  // Injected SetGlobalTx fields, not validated by signAppData.
  value: {
    isInternalTx: true,
    internalTXType: 4, // InternalTXType.ApplyChangeConfig
    change: {
      cycle: 1,
      change: {
        mode: "debug",
      },
    },
  },
  when: Date.now(),
  source: "1000000000000000000000000000000000000000000000000000000000000001",
}
```

Since the only requirement for `signAppData` to correctly execute,\
is to have just **one** validator node with staked coins in the whole network,\
this makes exploitation trivial for any active participating validator node.\
Furthermore, `signAppData` can be called continuously with different objects,\
allowing multiple steps to be chained to achieve higher impact.

## Impact Details

Since implementing exact validation of payloads in all other Shardeum/Shardus endpoints\
is simply impossible, there are guaranteed to be endpoints which will always be affected\
by this vulnerability. For example, the `set-global` gossip route used in our initial\
report #33632 is still affected by this vulnerability, because it contains no validation\
that no unexpected fields are present in the signed request. Additionally, let us note\
that this is in fact a good practice, as it allows for backwards and forwards compatibility\
of differently versioned Shardeum nodes, so implementing strict validation of payloads\
in all endpoints is not a viable solution.

As such, all the impact details listed in our previous report are still valid,\
and the `binary_sign_app_data` method can be used to forge a signed `GlobalAccounts`\
transaction to modify the network parameters, enable debug mode, change dev keys, etc.

## References

* Our (Neplox) report #33632: <https://bugs.immunefi.com/dashboard/submission/33632>
* Report #33632 archived on Immunefi's GitHub: <https://github.com/immunefi-team/Bounty\\_Boosts/blob/main/Shardeum%20Core/33632%20-%20%5BBC%20-%20Critical%5D%20Signature%20forgery%20on%20behalf%20of%20other%20nodes%20lead....md>
* Report #34484 by @ZhouWu: <https://github.com/immunefi-team/Bounty\\_Boosts/blob/main/Shardeum%20Core/34484%20-%20%5BBC%20-%20Critical%5D%20Tricking%20legit%20node%20to%20signed%20maliciously%20contr....md>
* Current `signAppData` implementation in `dev` branch: <https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/index.ts#L6573>

## Link to Proof of Concept

<https://gist.github.com/renbou/f77bb60fbcd421329a72b5185e0068a4>

## Proof of Concept

## Proof of Concept

To avoid writing up the same PoC as in report #33632, we will simply describe the\
modifications would need to be made in order to make the exploit work now.\
Do note, however, that currently we were unable to build a complete runnable PoC,\
as the Shardeum node (<https://github.com/shardeum/shardeum/tree/dev>) configured\
with the Core (<https://github.com/shardeum/core/tree/dev>) on the `dev` branch\
does not work with the configurations we tried: staking transactions do not go through\
due to most nodes erroring out with the `stuck_in_consensus_3` error.\
We have reported this issue to the Shardeum team in the competition Discord,\
and have requested a configuration setup with which transactions can be successfully run on the local network. Once these details are provided, we can update the PoC to have it\
run on the latest version of the Shardeum node and core.

## Local Shardeum network setup

The setup is pretty much the same as before, however now Shardeum uses\
NodeJS version v18.19.1, so the following `asdf` (version 0.16) commands can be used to install it:

```bash
asdf install nodejs v18.19.1
asdf set nodejs v18.19.1
```

Additionally, the `network-32.patch` file (<https://gist.githubusercontent.com/renbou/f77bb60fbcd421329a72b5185e0068a4/raw/92a27dee87379c1bca0614deed94d78f9d31aab8/network-32.patch>) now includes a change to modify\
the new `flexibleRotationDelta` setting to `0`, as suggested by `mgthura` from the Shardeum team in the Discord server. Without this change, nodes immediately start rotating once the network reaches `processing` mode, which makes testing more difficult.

To use the Shardeum node with the latest `core` version from the `dev` branch,\
we used `npm link` as specified in the `core` README: <https://github.com/shardeum/core/blob/dev/README.md>.

## JSON-RPC API setup

The setup of the JSON-RPC API only differs in the NodeJS version as well.

## Malicious node setup

The malicious node setup is the same, but should also use the new NodeJS version and new `network-32.patch` file. The `malicious-node.patch` does not change.

## Proof of Concept exploit

Currently, the `modify-config-poc.js` (<https://gist.github.com/renbou/f77bb60fbcd421329a72b5185e0068a4/raw/54d69c433ea8cf0a202d2a620cae8c86f61baa47/modify-config-poc.js>) attached to the GitHub Gist contains\
a slightly updated version of the original exploit with new dependencies (<https://gist.github.com/renbou/f77bb60fbcd421329a72b5185e0068a4/raw/54d69c433ea8cf0a202d2a620cae8c86f61baa47/package.json>) and\
fixes to make the staking code work with the new Shardeum node version.\
However, the `sign-app-data` endpoint call has not been updated to the`binary_sign_app_data` internal endpoint, as we are unable to update this\
part until the transaction issues are resolved.

This PoC can be ran to test the staking transaction issue, which manifests\
itself after the PoC prints the "STAKE TX CONFIRMED" message.\
Despite the transaction being confirmed on some nodes, the network disagrees\
on the correct state of the staker (`0xe6e789891Aad9E4ea1e0E37214Bd7067598BAdEc`) and node (`b9d86f47c6a9a5bee5b1399229c5493629a741d27cfed85aad04279d246e8538`) accounts:

Response from `:9002` node:

```bash
❯ curl http://localhost:9002/account/0xe6e789891Aad9E4ea1e0E37214Bd7067598BAdEc | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   681  100   681    0     0   147k      0 --:--:-- --:--:-- --:--:--  166k
{
  "account": {
    "balance": "84990000000000000000",
    "codeHash": "0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470",
    "nonce": "1",
    "operatorAccountInfo": {
      "certExp": 0,
      "lastStakeTimestamp": 1739105208481,
      "nominee": "b9d86f47c6a9a5bee5b1399229c5493629a741d27cfed85aad04279d246e8538",

...
```

Response from `:9003` node:

```bash
❯ curl http://localhost:9003/account/0xe6e789891Aad9E4ea1e0E37214Bd7067598BAdEc | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   249  100   249    0     0  87892      0 --:--:-- --:--:-- --:--:--  121k
{
  "account": {
    "balance": "100000000000000000000",
    "codeHash": "0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470",
    "nonce": "0",
    "operatorAccountInfo": null,
    "storageRoot": "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421"
  }
}
```

Nevertheless, we will describe the modifications which would need to be made\
once the transaction issues are resolved. Since the `sign-app-data` endpoint\
is now disabled, the `ask` call should be replaced with an `askBinary` call\
using the `serializeSignAppDataReq` serializer. It shouldn't be an issue\
to copy the necessary parts of the `askBinary` implementation from `p2p/Comms.ts` (<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/p2p/Comms.ts#L444>) and `network/index.ts` (<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/network/index.ts#L425).\\>
The `serializeSignAppDataReq` serializer can be taken from <https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/types/SignAppDataReq.ts#L17>.

Since no new validation mechanisms have been implemented anywhere in `signAppData` or `binary_sign_app_data` endpoints, everything else should just work.

If there is a need to test the exploit again, we can take the time to update it\
once the staking transaction issue is resolved. However, being completely honest,\
we do not see the need in doing so, as there are really no important differences\
between the old plaintext `sign-app-data` endpoint and the binary `binary_sign_app_data`.
