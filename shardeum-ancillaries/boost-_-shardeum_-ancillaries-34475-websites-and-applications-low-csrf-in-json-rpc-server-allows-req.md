# Boost \_ Shardeum\_ Ancillaries 34475 - \[Websites and Applications - Low] CSRF in Json RPC Server allo

Submitted on Tue Aug 13 2024 11:15:51 GMT-0400 (Atlantic Standard Time) by @neplox for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #34475

Report type: Websites and Applications

Report severity: Low

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Injecting/modifying the static content on the target application without JavaScript (reflected), such as: Reflected HTML injection, Loading external site data

## Description

## Brief/Intro

There's no CSRF protection in JRS which allows an attacker to make API requests on behalf of authenticated user. An attacker can modify certain config properties as well as manipulate log files.

## Vulnerability Details

A developer may request `/:passphrase` endpoint to get an `access_token`. This token is used for the `/log` api. The application doesn't have any CSRF protection so its possible to request the authenticated APIs on behalf of authenticated user.

## Impact Details

An attacker can delete important logs & change `recordTxStatus` and `statLog` config properties. Furthermore, by enabling these config properties, an attacker can exploit the SQL injection vulnerability (see \[SQL injection in json-rpc-server within the`txStatusSaver` function via the IP argument leads to application shutdown] report).

## References

<https://github.com/shardeum/json-rpc-server/blob/c3c462a4b18bc7517e086ff70f08ae6afede3b31/src/routes/authenticate.ts#L15C5-L15C8>

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

Go to `http://127.0.0.1:8080/authenticate/sha4d3um` to acquire an `access_token`

make an HTML page with the following content:

```
<script> location="http://127.0.0.1:8080/log/startRPCCapture" </script>
```

Open the page and see `message "RPC interface recording enabled"` which confirms that we were able to change JRS settings.
