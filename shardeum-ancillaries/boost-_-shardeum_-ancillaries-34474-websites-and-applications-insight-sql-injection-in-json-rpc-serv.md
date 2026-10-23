# Boost \_ Shardeum\_ Ancillaries 34474 - \[Websites and Applications - Insight] SQL injection in json-rp

Submitted on Tue Aug 13 2024 11:11:30 GMT-0400 (Atlantic Standard Time) by @neplox for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #34474

Report type: Websites and Applications

Report severity: Insight

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Taking down the application/website

## Description

### Brief/Intro

SQL injection in log saving functionality allows an unauth attacker to take down the json rpc server.

### Vulnerability Details

When `recordTxStatus` flag in the config set to true, the application will save transaction information to the sqlite3 db for logging purposes. The vulnerability lies in the txStatusSaver function. It takes an IP address from the tx and inserts it in to the db without the sanitization of special characters. Developers can enable `recordTxStatus` by opening `http://127.0.0.1:8080/log/startRPCCapture` page.

```
function txStatusSaver() {
...
let { txHash, injected, accepted, reason, type, to, from, ip, timestamp, nodeUrl } = txs[0]

...
      let placeholders = `NULL, '${txHash}', '${type}', '${to}', '${from}', '${injected}', '${accepted}', '${reason}', '${ip}', '${timestamp}', '${nodeUrl}'`
      let sql = 'INSERT OR REPLACE INTO transactions VALUES (' + placeholders + ')'

...
}
```

This function saves transactions that are received after the execution either `eth_sendInternalTransaction` or `eth_sendRawTransaction` methods.

The arguments of the abovementioned methods are passed to the `injectWithRetries(txHash, internalTx, args)` function that calls another function - `injectAndRecordTx(txHash, tx, args)`

In the end, the `injectAndRecordTx` function, calls `recordTxStatus`:

```
recordTxStatus({
  txHash,
  raw,
  injected: true,
  accepted: injectResult.success,
  reason: injectResult.reason || '',
  timestamp: tx.timestamp || Date.now(),
  ip: args[1000], // this index slot is reserved for ip, check injectIP middleware
  nodeUrl: baseUrl,
})
```

The `recordTxStatus` function later puts the transaction information to the `txStatuses` array from where our data ends up in the sqli vulnerable `txStatusSaver` function.

The problem with this call is, the application assumes that IP of the user is 1000th argument, but this is only partially true. From the injectIP.ts we can see the following middleware:

```
const injectIP = (req: Request, res: Response, next: NextFunction): void => {
  if (req.body.method === 'eth_sendRawTransaction' && CONFIG.recordTxStatus) req.body.params[1000] = req.ip
  next()
  return
}
```

The problem is, it adds IP only to the `eth_sendRawTransaction` call, ignoring the `eth_sendInternalTransaction` method. This is why its possible to provide IP value by ourselves and ultimately execute an SQLI vulnerability.

### Impact Details

* Take down the json rpc server by inserting big chunks of data and filling up all the RAM.
* Fill up all the available memory in the Secondary Storage.
* Comment out the transactions of other users so that transactions db won't be populated

## Proof of concept

### Set up shardeum network

Clone the Shardeum repo.

```
git clone https://github.com/shardeum/shardeum.git
cd shardeum
```

Switch to NodeJS 18.16.1, which is the version used by Shardeum in dev.Dockerfile and its various library requirements. For example, using asdf (<https://asdf-vm.com/>):

```
asdf install nodejs 18.16.1
asdf local nodejs 18.16.1
```

or

```
nvm use 18.16.1
```

Apply the debug-10-nodes.patch for network setup.

```
git apply debug-10-nodes.patch
```

Install dependencies and build the project.

```
npm ci
npm run prepare
```

Launch the network with 10 nodes as specified in the patch using the shardus tool.

```
npx shardus create 10
```

### Set up json rpc server

wait for a bit

Install json rpc server

```
git clone https://github.com/shardeum/json-rpc-server
```

Decrease the maxTxCountToStore to 1 in src/api.ts. This parameter defines how many transactions will be stored in the txStatuses array before they end up in sqlite3 db. For demonstration purposes, in order to execute the requests straight away, we change the parameter.

Switch to NodeJS 18.16.1, which is the version used by Shardeum in dev.Dockerfile and its various library requirements. For example, using asdf (<https://asdf-vm.com/>):

```
asdf install nodejs 18.16.1
asdf local nodejs 18.16.1
```

or

```
nvm use 18.16.1
```

then

```
npm ci
npm run start
```

If you see an error No Healthy Archivers in the Network. Terminating Server but the shardus network started successfully, wait for a few minutes for archive to became available and repeat the last command.

### Exploit

#### Preparation

Open `http://127.0.0.1:8080/authenticate/sha4d3um` to get an access token.

If the administrator likes the stats, they've opened `http://127.0.0.1:8080/log/startTxCapture` and enabled the capturing of transactions by themselves.

If it isn't the case, they can be tricked to open a page with the following code:

```
<script> location="http://127.0.0.1:8080/log/startRPCCapture" </script>
```

> More information about why this works is in the \[CSRF in Json RPC Server allows requesting authenticated API endpoints] report.

#### Run

Curl command and the whole exploit HTTP request can be found in <https://gist.github.com/Sh1Yo/2171d9d5271f39a27baf8e50b9ec8613>

After the request execution,

The application will hung up for several minutes, trying to insert all the data to the db. In case you have less than 10GB of available memory, the system will be forced to kill the process because there will be a lack of available RAM.

If you have more than 10GB of available memory, you can either repeat more insertions (the pattern is easily noticeable), or execute something like `stress --vm-bytes 10G --vm-keep -m 1` (<https://linux.die.net/man/1/stress>, available in the repositories of most linux distributions) to limit the amount of the available memory before request execution.
