# #39434 \[W\&A-Critical] Improper serialization can create an out-of-memory (OOM) issue on the archive

**Submitted on Jan 30th 2025 at 10:23:46 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39434
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

The Archiver server is utilizing the vulnerable safeStringify function from the shardus/types library for JSON serialization. This vulnerability allows attackers to supply malicious data that can cause the function to allocate a significant amount of memory on the server.

## Vulnerability Details

Most of the time, the archive server utilizes the safeStringify function from the shardus/types library. The vulnerable portion of this function is highlighted below:

```
          switch (options.bufferEncoding) {
            case 'base64':
              return JSON.stringify({
                value: Buffer.from(val['data']).toString('base64'),
                dataType: 'bb',
              })
          }
```

The Buffer.from method can be invoked with an object. If the object contains a length property, it will allocate a buffer with the specified length size. For example, Buffer.from({ length: 10 }) creates a buffer of 10 bytes, typically initialized with zeros. An attacker can exploit this behavior by providing excessively large numbers for buffer allocation, which can consume all available memory on the archive server. This can lead to an out-of-memory (OOM) condition.

## Impact Details

An OOM condition can render the archive server unresponsive, legitimate processes on the server may also be affected, leading to degraded performance and finally termination by os.

## References

<https://github.com/shardeum/lib-types/blob/ebf34c1538e8ece8a6e022cf4774ee27fd4ddcc7/src/utils/functions/stringify.ts#L122>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/c19ee4b28d9d813c1503227cdfb15af7>

## Proof of Concept

## Proof of Concept

I utilized a malicious validator as an attack vector against the archiver. To streamline testing, it is recommended to compile all nodes with the attack patch and activate the attack only on the node connected to the archiver. I developed an API method to activate the attack. Follow the steps below to reproduce the vulnerability:

1. Apply patch to core repository and build validator server.
2. Launch the network and wait until all nodes are active and the network is processing transactions.
3. Check the main.log file on the archive server for a message similar to: New Socket Connection.... :
4. Execute the following command to trigger the attack:

```shell
curl -X POST http://<IP>:<Port>/bomb
```

5. Observe the archiver's memory usage. The memory consumption should start increasing, causing the archiver to become unresponsive and eventually be terminated by the operating system due to an out-of-memory (OOM) condition.
