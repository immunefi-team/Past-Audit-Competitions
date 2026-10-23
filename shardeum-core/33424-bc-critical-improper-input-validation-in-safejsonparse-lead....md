# 33424 - \[BC - Critical] Improper input validation in safeJsonParse lead...

## Improper input validation in safeJsonParse leads to DOS and total network shutdown

Submitted on Jul 20th 2024 at 06:53:27 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33424

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://immunefi.com>

Impacts:

* Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
* Direct loss of funds
* Network not being able to confirm new transactions (total network shutdown)

### Description

### Brief/Intro

Shardeum validator nodes implemented in the <https://github.com/shardeum/shardeum> repository are vulnerable to complete DOS due to lack of input validation in `Utils.safeJsonParse` from the `@shardus/types` package, which is used for parsing messages received on all internal endpoints PRIOR to authorization, some external endpoints, and overall across the validator node implementation. Exploitation leads to a complete stall of all of the validator node's processes, and due to the simplicity of the exploit, it is possible to execute it on all nodes, including standby, simultaneously, shutting down the entire network. A more sophisticated attack would involve shutting down only a large part of the nodes, but not all, for attacker-controlled nodes to be the only ones available, which can then be used to overtake the whole network. Since the vulnerability can be triggered through any request via the internal protocol, and some external endpoints, it is difficult to block an incoming attack.

### Vulnerability Details

The `Utils.safeJsonParse` function from the `@shardus/types` package, which is commonly used throughout both `shardeum` and `shardus-core` repositories, does not perform validation on the passed data, expecting values with the `dataType` property set to `bb` or `u8ab` to have a string-typed `value` field. It is possible, however, to pass an "Array-like" object as the `value` field, which is accepted by `Buffer.from` used by the `typeReviver` function from `safeJsonParse`.

Since "Array-like" objects only require a `length` property to be set, it is easy to pass such an object from an incoming request, in which case `Buffer.from({length: x}, 'base64')` would attempt to sequentially copy `x` values from the "Array-like" object, each of which will be 0 due to no actual array values being set, making for an easy way to greatly increase the memory usage of the validator node without sending a large request, and cause a DOS of the validator node because the copying done by `Buffer.from` linearly depends in CPU time on the value of `x`. This can easily be checked in a `node` shell, using the same version that is used by `shardeum`, v18.16.1:

```js
node
Welcome to Node.js v18.16.1.
Type ".help" for more information.
> function time(f) {console.time('f');f();console.timeEnd('f')}
undefined
> time(() => Buffer.from({length:1_000_000},'base64'))
f: 49.438ms
undefined
> time(() => Buffer.from({length:10_000_000},'base64'))
f: 393.477ms
undefined
> time(() => Buffer.from({length:100_000_000},'base64'))
f: 4.013s
```

The most effective way this could be exploited is through the internal protocol, implemented in `@shardus/net`, which receives a `customJsonParser` option during initialization, which is set to `Utils.safeJsonParse` in `shardus-core/src/network/index.ts` (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/network/index.ts#L192>). The `customJsonParser` option value is then used for parsing each incoming internal protocol message without any prior validation (<https://github.com/shardeum/lib-net/blob/2832f1d4c92a3efb455239f146567f21fd80e4cb/src/index.ts#L368>):

```js
const extractUUIDHandleData = (
  augDataStr: string,
  remote: RemoteSender,
  header?: AppHeader,
  sign?: Sign
) => {
  let augData: AugmentedData = jsonParse(augDataStr, opts.customJsonParser)
```

Where `jsonParse` is a wrapper which calls `Utils.safeJsonParse` in our case since the option is set to it (<https://github.com/shardeum/lib-net/blob/2832f1d4c92a3efb455239f146567f21fd80e4cb/src/util/Encoding.ts#L5>):

```js
export const jsonParse = <T>(
  data: string,
  customParser?: (data: string) => T
): T => {
  return customParser ? customParser(data) : JSON.parse(data);
};
```

Sending a simple message of the form `{"dataType": "bb", "value": {"length": 1000000000}}` through the internal protocol will cause any validator node, no matter its current state and status in the network, to start allocating lots of memory and wasting CPU cycles on the `Buffer.from` initialization. The `length` value here can be chosen by an attacker depending on the desired effect and duration of the DOS. Since this happens in the internal protocol message receival procedure, the attacker does not even need to be authorized and part of the validator node network. Because `Utils.safeJsonParse` is used throughout the whole validator node implementation, this vulnerability can be exploited elsewhere as well, and must be fixed in the `safeJsonParse` implementation itself.

### Impact Details

The most basic outcome would be a total network shutdown caused by exploiting the DOS vulnerability on each available validator node, including the ones in standby. Sending large `length` values will cause nodes to either crash due to OOM, or spend long times in the `Buffer.from` call, which, when utilized across all nodes in the network, will halt all transaction processing and cause consensus mechanisms to crash later on. Because there are no prerequisites to the attack, the attacker can just keep on spamming DOS-causing requests to any endpoint which calls `Utils.safeJsonParse`, so that actually starting the network back up would require a fix to be deployed to all nodes, making it even more difficult to mitigate.

A more complex attack scenario, as said in the intro, would be the shutdown of nearly all, but not all, validator nodes, so that attackers nodes would be the only ones left available in the standby state. After the attackers nodes are picked as active and join the validators, the remaining honest validators can be shut down using the DOS vulnerability, which would leave the whole network to be controlled by the attacker. This means the network will continue functioning, but the attacker will be able to execute any transaction they want, and drain all the funds to themselves.

### References

* `typeReviver` (used by `Utils.safeJsonParse`) implementation in `@shardus/types` package: <https://github.com/shardeum/lib-types/blob/3da9a31c38bdc4213b824114607c7a4a6e10ad6c/src/utils/functions/stringify.ts#L184>
* `Buffer.from` with "Array-like" parameter functionality documentation: <https://nodejs.org/api/buffer.html#static-method-bufferfromarray>

## Proof of concept

This POC was written in order to demonstrate total network shutdown as the main impact, as fund loss would be an impact that would follow due to all validator nodes except the attacker's being crippled. In order to simplify the POC only the main impact of validator node DOS is implemented.

`internal-dos.js` contained in the attached gist (<https://gist.github.com/renbou/bf18942a4db8d41edbbbf73240eb94d4>) contains the main exploit code for automatically disabling all the active and standby nodes of the network after retrieving their addresses from the archiver. This writeup is present just to showcase exactly how it was tested and how it works.

**WARNING!** the exploit causes **each** validator node to slowly accumulate up to a gigabyte (the `1_000_000_000` constant in the exploit) worth of memory, so make sure to run it in an unimportant environment and be ready for possible crashes and OOM errors. The value `1_000_000_000` can also be decreased for running the setup and exploit locally, it was chosen in order to clearly demonstrate the impact of node DOS via memory and CPU resource consumption.

### Local Shardeum network setup

This vulnerability is equally exploitable with any number of nodes as exploitation of a single validator node requires sending just a single message via the internal protocol or to any external endpoint which uses the `Utils.safeJsonParse` method. For demonstration purposes, however, a network with only 32 validator nodes is created.

Any Shardeum network using the current validator node code will be vulnerable, so it is not necessary to follow these exact steps. They are here just to showcase how the POC was tested by me.

1. Clone the Shardeum repo and switch to the last commit on the `dev` branch, which is `c7b10c2370028f7c7cbd2a01839e50eb50faa904` as of this POC's submission.

   ```bash
   git clone https://github.com/shardeum/shardeum.git
   cd shardeum
   git switch --detach c7b10c2370028f7c7cbd2a01839e50eb50faa904
   ```
2. Switch to NodeJS 18.16.1, which is the version used by Shardeum in `dev.Dockerfile` and its various library requirements. For example, using asdf (<https://asdf-vm.com/>):

   ```bash
   asdf install nodejs 18.16.1
   asdf local nodejs 18.16.1
   ```
3. Apply the `32-nodes.patch` file from the attached gist (<https://gist.github.com/renbou/bf18942a4db8d41edbbbf73240eb94d4>) for network setup. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

   ```bash
   git apply 32-nodes.patch
   ```
4. Install dependencies and build the project.

   ```bash
   npm ci
   npm run prepare
   ```
5. Launch the network with 32 nodes as specified in the patch using the `shardus` tool.

   ```bash
   npx shardus create 32
   ```

After this step, 15-20 minutes are required as usual for at least some validator nodes to go into being active, at which point the exploit itself can be ran. I used the <http://localhost:3000/> monitor to wait for nodes to start activating.

### DOS exploitation for network shutdown

As said in the introduction, `internal-dos.js` from the attached gist (<https://gist.github.com/renbou/bf18942a4db8d41edbbbf73240eb94d4>) contains the exploit code and can be ran on any network as long as a valid archiver URL is suplied via the `ARCHIVER_URL` variable at the top of the script. It can be ran using `NodeJS`, and only requires the `@shardus/net` and `axios` libraries to be installed, as specified in the attached `package.json` (<https://gist.github.com/renbou/bf18942a4db8d41edbbbf73240eb94d4>). Following is a detailed writeup of how it works:

1. Multiple requests to the specified archiver's `full-nodelist` endpoint are made in order to retrive the full list of validator nodes in the network, including standby, syncing, and active nodes. Running the DOS on these nodes means that no more nodes will be available, leading to a total shutdown of the network.
2. Each node's `/nodeinfo` endpoint is queried to retrieve the `internalIp` and `internalPort` values, which host the internal protocol of the node, used in this exploit of the vulnerability.
3. A message of the form `{ dataType: "bb", value: { length: 1_000_000_000 } }` is sent using the internal protocol without any headers and metadata. It will be directly passed to `Utils.safeJsonParse`, triggering the vulnerability. `Buffer.from` will then block NodeJS' event loop, making each validator node unresponsive and practically stopping the network.

Shardeum's monitor dashboard, launched by default on <http://localhost:3000/> can be used to check that all active nodes will be now marked as red as they go offline and stop reporting to the monitor. Making any request to the validator nodes will hang, as the network is completely stopped at this point.
