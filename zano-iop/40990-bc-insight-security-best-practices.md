# #40990 \[BC-Insight] Security best practices

**Submitted on Mar 8th 2025 at 18:30:45 UTC by @jovi for** [**IOP | Zano**](https://immunefi.com/audit-competition/iop-zano)

* **Report ID:** #40990
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/zano-iop/tree/main/src/p2p/net\\_node.inl>
* **Impacts:**
  * Security best practices
  * QA Report

## Description

As spoken with the project at the discord channel, this submission contains findings that are not contained in the scope of impacts of this program and don't represent medium/high/critical severity vulnerabilities.

## 01

**Summary**\
In the P2P node server’s peer management code (`peerlist_manager` and `node_server`), two related flaws allow attackers to abuse the peerlist trimming logic and manipulate peer IP classifications. Specifically:

1. The `trim_white_peerlist()` and `trim_gray_peerlist()` functions are implemented incorrectly, causing the wrong lists to be trimmed and possibly allowing untrusted or stale entries to remain in the white peerlist.
2. When receiving peer information from an incoming connection, the code trusts the incoming IP/Port too readily (and does not cross-check it sufficiently), opening the door for peer IP spoofing or incorrect classification.

If exploited, these bugs can result in a peerlist that is improperly populated or trimmed, leading to disruptive network behavior (e.g., spurious connections, partial DoS, or peer partitioning).

***

## Vulnerability Details

### Location

1. **peerlist\_manager::trim\_white\_peerlist & trim\_gray\_peerlist**
   * Located in `net_peerlist`.
   * Code snippet reference (showing incorrect trimming):

     ```cpp
     // @audit -> trims the wrong list on both methods
     inline void peerlist_manager::trim_white_peerlist()
     {
       CRITICAL_REGION_LOCAL(m_peerlist_lock);
       while(m_peers_gray.size() > P2P_LOCAL_GRAY_PEERLIST_LIMIT)
       {
         peers_indexed::index<by_time>::type& sorted_index = m_peers_gray.get<by_time>();
         sorted_index.erase(sorted_index.begin());
       }
     }

     inline void peerlist_manager::trim_gray_peerlist()
     {
       CRITICAL_REGION_LOCAL(m_peerlist_lock);
       while(m_peers_white.size() > P2P_LOCAL_WHITE_PEERLIST_LIMIT)
       {
         peers_indexed::index<by_time>::type& sorted_index = m_peers_white.get<by_time>();
         sorted_index.erase(sorted_index.begin());
       }
     }
     ```

     Both methods call erase on the *opposite* list, which inverts the trimming logic.
2. **node\_server::set\_peer\_just\_seen / handle\_remote\_peerlist**
   * Located in `net_node` code (`net_node.inl` ).
   * Code snippet reference (showing insufficient verification of incoming addresses):

     ```cpp
     bool node_server<t_payload_net_handler>::do_handshake_with_peer(peerid_type& pi, p2p_connection_context& context_, bool just_take_peerlist)
     ```

{\
...\
if(!handle\_remote\_peerlist(rsp.local\_peerlist, rsp.node\_data.local\_time, context))\
{\
// merges peers without verifying correctness thoroughly\
// ...\
}\
\`\`\`

```
    This insufficient cross-check can add arbitrary IP addresses to the white (trusted) list.
```

### Description

1. **Incorrect List Trimming**\
   The `trim_white_peerlist()` function *actually* trims the gray list, and `trim_gray_peerlist()` function *actually* trims the white list. This inverted logic can cause a scenario where:
   * The white list grows without correct pruning, potentially filled with malicious or stale entries.
   * The gray list is pruned too aggressively (or not at all), impacting how new or less-trusted peers are recognized.
2. **IP Spoofing / Insufficient Verification**\
   When new remote peer information arrives via an incoming connection, there is minimal cross-checking or confirmation that the provided IP addresses/ports are accurate. Attackers or misconfigured peers can inject bogus addresses into the white list, degrade the node's connectivity to honest peers, or trigger unwanted IP blocks.

### Impact

* **Inaccurate Peer Reputation**: The white list is supposed to represent more-trusted peers, but these issue lets unverified addresses end up in it.
* **Potential Node Resource Waste**: Large or invalid peer entries in the white list or forced IP blocks can waste resources and degrade performance.

## 02

**Summary**\
In `handle_request_get_objects()`, the error message incorrectly cites the number of blocks (`arg.blocks.size()`) even when the root cause is an excessive number of transactions (`arg.txs.size()`). This leads to misleading logs and hinders accurate troubleshooting.

***

**Vulnerability Details**\
**Location**

* File: `t_currency_protocol_handler<t_core>::handle_request_get_objects`
* Function: `handle_request_get_objects(...)`

**Description**\
When `arg.blocks.size()` or `arg.txs.size()` exceeds the respective maximum count, an error log is generated, and the connection is dropped. However, the log statement unconditionally prints `arg.blocks.size()` regardless of whether `arg.txs.size()` is responsible for the threshold breach. This creates confusion during debugging or monitoring because it obscures which specific parameter (blocks vs. transactions) caused the violation.

**Code Snippet**

```cpp
// Trimmed for clarity
if (arg.blocks.size() > CURRENCY_PROTOCOL_MAX_BLOCKS_REQUEST_COUNT ||
    arg.txs.size() > CURRENCY_PROTOCOL_MAX_TXS_REQUEST_COUNT)
{
  LOG_ERROR_CCONTEXT("Requested objects count is to big ("
                     << arg.blocks.size()
                     << ")expected not more then "
                     << CURRENCY_PROTOCOL_MAX_BLOCKS_REQUEST_COUNT);
  m_p2p->drop_connection(context);
}
```

The snippet demonstrates that only `arg.blocks.size()` is reported, even if the `arg.txs.size()` exceeded its limit.

**Impact**

* **Operational Misinterpretation**: Automated systems or operators may be misled by the incorrect reference in the log, delaying corrective actions or misdirecting investigations.

## 03

**Summary**\
The `connections_maker()` routine within `node_server<t_payload_net_handler>` conflates all outgoing connections as if they were white-listed (“white”) connections. This erroneous assumption leads to incorrect logic that can create either too few or too many white connections, undermining the intended networking and connection policies.

***

## Vulnerability Details

### Location

`node_server<t_payload_net_handler>::connections_maker()` in the method snippet:

```cpp
size_t expected_white_connections = (m_config.m_net_config.connections_count * P2P_DEFAULT_WHITELIST_CONNECTIONS_PERCENT) / 100;

// @audit -> wrongly assumes those to be white_connections
size_t conn_count = get_outgoing_connections_count();
if (conn_count < m_config.m_net_config.connections_count)
{
  // @audit since the idle_worker will run this multiple times, the second iteration will count
  // @audit grey connections at the conn_count
  if (conn_count < expected_white_connections)
  {
    //start from white list
    if (!make_expected_connections_count(true, expected_white_connections))
      return false;
    //and then do grey list
    if (!make_expected_connections_count(false, m_config.m_net_config.connections_count))
      return false;
  }
  else
  {
    //start from grey list
    if (!make_expected_connections_count(false, m_config.m_net_config.connections_count))
      return false;
    //and then do white list
    if (!make_expected_connections_count(true, m_config.m_net_config.connections_count))
      return false;
  }
}
```

### Description

The code tracks a total count of outgoing connections (`conn_count`) and applies that figure as though all these connections are white connections. In reality, the method `get_outgoing_connections_count()` can also return connections established from other categories (for example, “grey” connections), inflating the count.

* **Root cause:** The logic fails to distinguish white connections from grey connections when deciding whether to create more white connections.
* **Deviation from intended behavior:** The system tries to maintain a certain number of white connections but incorrectly uses the overall outgoing connection count, leaving the node with insufficient white connections.

### Impact

1. **Connection Policy Issues:** The node may fail to achieve the desired count of genuine white connections, hurting network health or reliability.

## Proof of Concept

## 01

1. **Wrong Peerlist Trimming**
   * Run a local node that restarts repeatedly when the white list is at or near capacity.
   * Notice that the white list does *not* get trimmed, but the gray list entries do—showing the mismatch in code and actual effect.
2. **IP Spoofing**
   * Create a malicious peer that, during handshake, sends a fake peer list with random or malicious IP addresses.
   * Observe that the node merges these without verifying if the addresses correspond to the IP of the handshake creator.

## 02

1. Create a peer request with `arg.blocks.size()` within limits but `arg.txs.size()` exceeding `CURRENCY_PROTOCOL_MAX_TXS_REQUEST_COUNT`.
2. Observe in logs that the error message only mentions blocks, showing a misleading reference to `arg.blocks.size()` being “too big”.

## 03

1. **Start a node** configured to have a limited number of white connections (for example, `connections_count=10`, with `P2P_DEFAULT_WHITELIST_CONNECTIONS_PERCENT=50`).
2. **Allow the node** to discover other grey peers before any white peers are available.
3. **Observe** that `get_outgoing_connections_count()` grows to or above `expected_white_connections` with grey connections, preventing the node from opening any new white connections. Despite the code’s intention to ensure at least 50% white connections, the final state mostly has grey connections.
