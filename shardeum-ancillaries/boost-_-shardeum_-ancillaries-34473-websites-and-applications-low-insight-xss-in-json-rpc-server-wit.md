# Boost \_ Shardeum\_ Ancillaries 34473 - \[Websites and Applications - Low] Insight XSS in json rpc serv

Submitted on Tue Aug 13 2024 11:06:09 GMT-0400 (Atlantic Standard Time) by @neplox for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #34473

Report type: Websites and Applications

Report severity: Low

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Redirecting users to malicious websites (open redirect)
* Injecting/modifying the static content on the target application without JavaScript (reflected), such as: Reflected HTML injection, Loading external site data

## Description

## Brief/Intro

To send error information on the `/log/tsx` endpoint, the application uses `res.send()` function that returns HTML response. It's possible to cause an error in sqlite3 query by providing a query with null byte. The error message would return all the provided data so we have a reflected html injection.

## Vulnerability Details

We can use any parameter that goes into the DB query, such as `ip`. The value will be inserted in to the `sqlFilter` and `sqlString` variables. It'll ultimately end up in `const txs = db.prepare(sqlString).all()` function that'll return an error message containing the value we've provided.

## Impact Details

The vulnerability can be used against the administrators of a json rpc node. Using html injection its possible to trick them into execution of requests that may affect the JRS settings by making redirects or creating forms.

## References

<https://github.com/shardeum/json-rpc-server/blob/d799a64c1ab4a7cffdf472a8be689fe7afb993e9/src/routes/log.ts#L320C1-L320C38>

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

Open `http://127.0.0.1:8080/authenticate/sha4d3um` to get an access token.

Go to `http://127.0.0.1:8080/log/txs?ip=<s>html</s>%00` to see an html injection
