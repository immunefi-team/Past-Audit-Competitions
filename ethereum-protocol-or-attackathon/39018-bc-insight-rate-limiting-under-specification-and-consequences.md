# #39018 \[BC-Insight] Rate Limiting Under-Specification and Consequences

**Submitted on Jan 20th 2025 at 13:52:20 UTC by @csludo for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #39018
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ethereum/consensus-specs>
* **Impacts:**
  * Unintended chain split affecting less than 25% of the network (Network partition)

## Description

## Brief/Intro

The rate limiting mechanism in current implementations is under-specified, leading to inconsistent behavior and unintended consequences across clients. This lack of clarity results in network instability, degraded performance, and forks, as demonstrated below.\
For example, an honest lighthouse client will disconnect from an honest prysm client because they chose different rate-limiting parameters and have different rate-limiting error codes.

## Vulnerability Details

Rate limiting is a critical component of such a decentralized network, ensuring that nodes do not get overwhelmed by misbehaving peers, malicious actors, or excessive traffic. Such a component needs to be well-defined and consistently implemented across clients, to avoid incompatible behaviors and incompatibility.

Rate limiting, as implemented across various clients, relies on generic algorithms such as the leaky bucket model. However, the specification provides insufficient detail about key parameters and their expected behavior, resulting in:

* **Ambiguity in Configuration**: Different clients implement varying rates and limits, leading to inconsistency in how rate limiting is applied.
* **Lack of Standardization**: Critical aspects, such as whether limits should be peer-specific or topic-specific, are not clearly defined.
* **Undefined Errors**: There is no guidance on how clients should handle nodes that temporarily exceed limits, leading to excessive disconnections. Additionally, peer score adjustments can misinterpret rate limiting errors as RPC errors, further exacerbating the problem. Standardizing error codes for rate limiting would help mitigate this issue.

This is especially problematic because these issues disproportionately affect well-connected peers with low latency. These peers are among the most important for maintaining network stability and efficient communication.

For example, Lighthouse clients frequently face disconnections when interacting with Prysm nodes due to Prysm's restrictive rate limiting configuration. The rpc-limiter-topic in Prysm only allows 5 request units per second (with a capacity of 10 units). When Lighthouse attempts to sync, its requests often exceed this limit, resulting in disconnections. Note, that this happens even though lighthouse is rate limiting its outgoing requests. This leads to an inability to maintain connectivity and highlights the need for standardized rate limiting practices across clients.The POC below demonstrates the impact of this under-specification and the resulting network fork.

## Impact Details

The under-specification of rate limiting has significant consequences:

* **Inconsistent Interoperability**: Clients implement varying thresholds, leading to unpredictable behavior when nodes communicate across implementations.
* **Unintended Disconnections**: Well-connected peers with low latency may face frequent disconnections, as they inadvertently exceed restrictive limits.
* **Network Partitioning**: Excessive disconnections can isolate nodes, particularly new joiners, from the network. This increases the risk of forks and reduces overall network stability.
* **Degraded Performance**: Disconnecting from well-connected peers negatively impacts propagation times and the efficiency of message delivery.

These issues are exacerbated when multiple clients with differing rate-limiting behaviors operate on the same network, further increasing the likelihood of partitions and degraded consensus.

## References

To highlight our point, we reference some code parts from different clients and how differently rate limiting is handled. The versions are the latest stable versions at time of writing.

### Prysm

Prysm does inbound and outbound rate-limiting:\
Inbound: <https://github.com/prysmaticlabs/prysm/blob/v5.2.0/beacon-chain/sync/rate\\_limiter.go#L65\\>
On Inbound rate-limiting it aggregates some topics together and also has a separate limit for all RPC requests of one peer:\
<https://github.com/prysmaticlabs/prysm/blob/v5.2.0/beacon-chain/sync/rate\\_limiter.go#L37\\>
The values for inbound are different than the lighthouse values. Here are some default values:\
<https://github.com/prysmaticlabs/prysm/blob/v5.2.0/cmd/beacon-chain/flags/base.go#L184-L193\\>
Outbound rate-limiting: <https://github.com/prysmaticlabs/prysm/blob/v5.2.0/beacon-chain/sync/initial-sync/blocks\\_fetcher.go#L136\\>
The prysm error message:\
<https://github.com/prysmaticlabs/prysm/blob/v5.2.0/beacon-chain/p2p/types/rpc\\_errors.go#L12>

### Lighthouse

Lighthouse does inbound and outbound rpc rate limiting:\
<https://github.com/sigp/lighthouse/blob/v6.0.1/beacon\\_node/lighthouse\\_network/src/rpc/mod.rs#L171\\>
Lighthouse uses different values compared to prysm:\
<https://github.com/sigp/lighthouse/blob/v6.0.1/beacon\\_node/lighthouse\\_network/src/rpc/config.rs#L102\\>
Lighthouse sends a different type of error message:\
<https://github.com/sigp/lighthouse/blob/v6.0.1/beacon\\_node/lighthouse\\_network/src/rpc/methods.rs#L603\\>
Lighthouse also separates the case of requests being too large:\
<https://github.com/sigp/lighthouse/blob/v6.0.1/beacon\\_node/lighthouse\\_network/src/rpc/mod.rs#L410>

### Lodestar

Lodestar does inbound rate-limiting per peer and protocolID:\
<https://github.com/ChainSafe/lodestar/blob/v1.25.0/packages/reqresp/src/rate\\_limiter/ReqRespRateLimiter.ts#L57\\>
It takes configuration values from Lighthouse:\
<https://github.com/ChainSafe/lodestar/blob/v1.25.0/packages/beacon-node/src/network/reqresp/rateLimit.ts#L7\\>
However, the link and these configuration values are outdated.\
Additionally, Lodestar also implements a total rate limiter across all connections (which is currently not used):\
<https://github.com/ChainSafe/lodestar/blob/v1.25.0/packages/reqresp/src/rate\\_limiter/ReqRespRateLimiter.ts#L15>

## Proof of Concept

## Proof of Concept

To emphasize the consequences of the under-specification, we provide a POC where the rate limiting between two clients results in the incapacity of two honest clients to communicate properly after a network event.

To replicate this issue, you can simulate a local network and introduce a temporary network split for a few minutes. This will force nodes to resynchronize, triggering the rate limiting behavior. We recommend using unmodified, stable versions of nodes for this test. The most efficient way to perform this test is by using attacknet (<https://github.com/crytic/attacknet>).

Set up attacknet with the repository's instructions and then create two config files:`test-suites/poc_config.yaml` and `network-configs/poc_net_config.yaml` with the content below and then run the simulation using `./attacknet start poc_config`.

During the simulation, you can observe the network with Dora (deployed with this setup). You will notice that the fork remains unresolved because the nodes fail to communicate properly due to rate limiting. To confirm that rate limiting is the root cause, repeat the same simulation without rate limits. Also, observing the logs shows the rate limiting from Prysm and the following peer score decrease (by Lighthouse) since it thinks it was an RPC error. All of this preventing the clients to maintain proper communication.

Note that this is happening even though lighthouse limits its outgoing requests, because, as argued above, configuration parameters are not standardized.

`test-suites/poc_config.yaml`:

```yaml
attacknetConfig:
  allowPostFaultInspection: true
  existingDevnetNamespace: kt-autorun-network-split
  grafanaPodName: grafana
  grafanaPodPort: 3000
  reuseDevnetBetweenRuns: true
  waitBeforeInjectionSeconds: 300
harnessConfig:
  networkConfig: poc_net_config.yaml
  networkPackage: github.com/ethpandaops/ethereum-package@e60afbeb7cefd1ee853c9bdca0041a6d4040fe78
  networkType: ethereum
testConfig:
  tests:
  - health:
      enableChecks: true
      gracePeriod: 60m0s
    planSteps:
    - chaosFaultSpec:
        apiVersion: chaos-mesh.org/v1alpha1
        kind: NetworkChaos
        spec:
          action: partition
          direction: both
          duration: 300s
          mode: all
          selector:
            labelSelectors:
              kurtosistech.com.custom/ethereum-package.partition: partA
          target:
            mode: all
            selector:
              labelSelectors:
                kurtosistech.com.custom/ethereum-package.partition: partB
      description: network split
      stepType: injectFault
    - description: wait for faults to terminate
      stepType: waitForFaultCompletion
    testName: network-split__prysm-lighthouse
```

`network-configs/poc_net_config.yaml`:

```yaml
additional_services:
- dora
- goomy_blob
- tx_spammer
- blob_spammer
- el_forkmon
- beacon_metrics_gazer
- prometheus_grafana
ethereum_metrics_exporter_enabled: true
global_log_level: info
network_params:
  deneb_fork_epoch: 0
  genesis_delay: 120
participants:
- cl_extra_labels:
    ethereum-package.partition: partA
  cl_image: gcr.io/prysmaticlabs/prysm/beacon-chain:v5.2.0
  cl_log_level: debug
  cl_max_cpu: 16000
  cl_max_mem: 16384
  cl_min_cpu: 100
  cl_min_mem: 256
  cl_type: prysm
  count: 2
  el_extra_labels:
    ethereum-package.partition: partA
  el_image: ethereum/client-go:v1.14.11
  el_log_level: debug
  el_max_cpu: 16000
  el_max_mem: 16384
  el_min_cpu: 100
  el_min_mem: 256
  el_type: geth
  vc_extra_labels:
    ethereum-package.partition: partA
  vc_log_level: debug
  vc_max_cpu: 3000
  vc_max_mem: 1028
  vc_min_cpu: 100
  vc_min_mem: 256
- cl_extra_labels:
    ethereum-package.partition: partA
  cl_image: gcr.io/prysmaticlabs/prysm/beacon-chain:v5.2.0
  cl_log_level: debug
  cl_max_cpu: 16000
  cl_max_mem: 16384
  cl_min_cpu: 100
  cl_min_mem: 256
  cl_type: prysm
  count: 2
  el_extra_labels:
    ethereum-package.partition: partA
  el_image: nethermind/nethermind:1.28.0
  el_log_level: debug
  el_max_cpu: 16000
  el_max_mem: 16384
  el_min_cpu: 100
  el_min_mem: 256
  el_type: nethermind
  vc_extra_labels:
    ethereum-package.partition: partA
  vc_log_level: debug
  vc_max_cpu: 3000
  vc_max_mem: 1028
  vc_min_cpu: 100
  vc_min_mem: 256
- cl_extra_labels:
    ethereum-package.partition: partB
  cl_image: sigp/lighthouse:v6.0.1
  cl_log_level: debug
  cl_max_cpu: 16000
  cl_max_mem: 16384
  cl_min_cpu: 1000
  cl_min_mem: 256
  cl_type: lighthouse
  count: 2
  el_extra_labels:
    ethereum-package.partition: partB
  el_image: ethereum/client-go:v1.14.11
  el_log_level: debug
  el_max_cpu: 16000
  el_max_mem: 16384
  el_min_cpu: 100
  el_min_mem: 256
  el_type: geth
  vc_extra_labels:
    ethereum-package.partition: partB
  vc_log_level: debug
  vc_max_cpu: 3000
  vc_max_mem: 1028
  vc_min_cpu: 100
  vc_min_mem: 256
- cl_extra_labels:
    ethereum-package.partition: partB
  cl_image: sigp/lighthouse:v6.0.1
  cl_log_level: debug
  cl_max_cpu: 16000
  cl_max_mem: 16384
  cl_min_cpu: 100
  cl_min_mem: 256
  cl_type: lighthouse
  count: 2
  el_extra_labels:
    ethereum-package.partition: partB
  el_image: nethermind/nethermind:1.28.0
  el_log_level: debug
  el_max_cpu: 16000
  el_max_mem: 16384
  el_min_cpu: 100
  el_min_mem: 256
  el_type: nethermind
  vc_extra_labels:
    ethereum-package.partition: partB
  vc_log_level: debug
  vc_max_cpu: 3000
  vc_max_mem: 1028
  vc_min_cpu: 100
  vc_min_mem: 256
- cl_extra_labels:
    ethereum-package.partition: partB
  cl_image: sigp/lighthouse:v6.0.1
  cl_log_level: debug
  cl_max_cpu: 16000
  cl_max_mem: 16384
  cl_min_cpu: 100
  cl_min_mem: 256
  cl_type: lighthouse
  count: 2
  el_extra_labels:
    ethereum-package.partition: partB
  el_image: nethermind/nethermind:1.28.0
  el_log_level: debug
  el_max_cpu: 16000
  el_max_mem: 16384
  el_min_cpu: 100
  el_min_mem: 256
  el_type: nethermind
  vc_extra_labels:
    ethereum-package.partition: partB
  vc_log_level: debug
  vc_max_cpu: 3000
  vc_max_mem: 1028
  vc_min_cpu: 100
  vc_min_mem: 256
- cl_extra_labels:
    ethereum-package.partition: partB
  cl_image: sigp/lighthouse:v6.0.1
  cl_log_level: debug
  cl_max_cpu: 16000
  cl_max_mem: 16384
  cl_min_cpu: 100
  cl_min_mem: 256
  cl_type: lighthouse
  count: 2
  el_extra_labels:
    ethereum-package.partition: partB
  el_image: nethermind/nethermind:1.28.0
  el_log_level: debug
  el_max_cpu: 16000
  el_max_mem: 16384
  el_min_cpu: 100
  el_min_mem: 256
  el_type: nethermind
  vc_extra_labels:
    ethereum-package.partition: partB
  vc_log_level: debug
  vc_max_cpu: 3000
  vc_max_mem: 1028
  vc_min_cpu: 100
  vc_min_mem: 256
persistent: true
snooper_enabled: true
```
