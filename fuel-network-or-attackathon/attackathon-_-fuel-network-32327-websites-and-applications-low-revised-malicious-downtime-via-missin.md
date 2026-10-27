# Attackathon \_ Fuel Network 32327 - \[Websites and Applications - Low] REVISED Malicious Downtime via

Submitted on Tue Jun 18 2024 12:09:41 GMT-0400 (Atlantic Standard Time) by @blackgrease for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32327

Report type: Websites and Applications

Report severity: Low

Target: <https://github.com/fuellabs/fuels-wallet/tree/4a20b3d8684a07e40bd2f7559032941b5adbe022>

Impacts:

* Taking down the application/website

## Description

\##REPORT FOREWORD

1. As requested in submission \[32310], this report is an improved version and provides the requested information along with a much better PoC to show case impact.
2. This is not a DDoS vulnerability as the attack comes from a single machine and uses a low number requests.
3. This is not a bug stemming from lack of rate-limiting but rather lack of input validation.
4. CloudFlare WAF does not mitigate this vulnerability as its protections are bypassed.
5. The backend GraphQL server is a critical asset connected to the Fuel Wallet Extension as the extension is unusable without it.

\##INTRODUCTION

The Fuel Wallet Chrome Extension maintains communication with a GraphQL server located on <https://testnet.fuel.network/v1/graphql> that can be forced to have malicious downtime due to missing input validation on queries, which results in the Fuel Wallet Browser Extension loosing its functionality.

\##DESCRIPTION

While the Fuel Wallet Chrome Extension is in use, it is in constant communication with a GraphQL server located at <https://testnet.fuel.network/v1/graphql>. This GraphQL server is used to load information in the Extension User Interface the moment the user opens it. Furthermore, every second, the Fuel Wallet queries this server; specifically using these queries, 'getChain', 'getNodeInfo', and 'getBalances'. A user triggered query is 'getTransactionsByOwner'. The GraphQL server also handles mutations and other queries in its schema. As such, the impact of this vulnerability affects the entire asset.

The GraphQL server has introspection enabled. A circular relationship exists in GraphQL’s built-in introspection system. Also known as recursive queries, circular queries occur when two nodes in a GraphQL schema are bidirectionally referenced using an edge. This circular reference can allow a client to build a complex query that forces the server to return an exponentially large response each time the query completes a “circle.” Therefore, when introspection is enabled AND does not have properly implemented protections on the introspection query, an attacker has access to create a resource intensive circular query right out of the gate. The GraphQL server has a query depth limit that rejects queries that have a depth greater than 32 from being executed but this protection has been set high.

Furthermore, the endpoint supports Alias Batching and it does not have any restrictions on the number of aliases that a query can use. A resource-intensive query is created by using: The circular-introspection query acts as a trigger (sent below the high query depth limit) and its impact is escalated by chaining it together with the missing restrictions on the number of aliases that can be used. As an attacker, I can utilize a cyclic introspection query (runs below the query depth limit) and use it in 15,000 aliases to create a resource-intensive query that negatively impacts the application by creating unexpected downtime which results in users of the Fuel Wallet Extension experiencing loss of the extensions functionality. There is no query cost analyzer implemented further allowing expensive queries to be executed.

Response times are used to show the impact of the queries on the performance of the server. As the complexity of the queries increases, so does the response time of the server. As the response times are high, this is a clear indicator that the servers performance is being affected.

In comparison, during testing, a single cyclic-introspection query takes a second to execute. The same cyclic-introspection query when used 15,000 times in an Alias Batch takes \~90 seconds to execute. As an attacker, by sending this resource-intensive query multiple times to the GraphQL endpoint, I can effectively result in its performance depreciating which results in malicious downtime affecting the Fuel browser extensions functionality. Malicious queries that contain more than 17,000 aliases receive 5xx Internal Server errors.

I have attached 3 different kinds of Proof-of-Concepts.

1. There are screenshots - specifically for queries using 5000, 10000, 15000 and 17000 aliases - showing the response times (bottom right corner) along with a confirmation on the number of aliases used. The number of aliases is shown by using a regex pattern "q\d" to show the total number used.
2. I have also attached a Python3 script in the gist link that sends increasingly complex queries and shows the increasing response times. A second code section is used, to execute the full attack sequence against the server. To demonstrate the impact straight away, the testing code has been commented out and the PoC starts execution from the attack phase. The gist link also contains the raw requests specifically for 5000, 10000, 15000 and 17000 aliases.
3. A small video link showing the loss of extension functionality when resource-intensive queries are being sent to the Backend GraphQL server using the Python3 PoC script.

During testing, if Burp is used, the response shown is "message is too large to be displayed". This is because of the large number of data being returned (389,039,396 bytes in the case of 15,000 aliases being used.). This is a GUI issue of Burp and unrelated to the bug. Using a GraphQL client may allow for responses to be seen.

The GraphQL server is behind Cloudflare WAF though the protections provided are negated as the exploitation phase uses a low-requests-per-second. Using the testing results, a resource intensive query, takes approximately 90 seconds to execute. This means that there is a 90 second time-frame where the servers resources are kept occupied. In this, 90 second window, more resource intensive queries can be sent to further occupy more server resources. As an attacker, by implementing a 10 second time delay in between requests (6 requests in 1 minute), 9 resource intensive queries can be sent that further consume resources. Overtime, the server becomes progressively overloaded and starts to fail legitimate user requests from the Fuel Wallet Extension causing users to loose some of the legitimate functionality in the extension. The high time-delay effectively bypasses the protections offered by Cloudflare. An attack can even use higher time-delays to make the attack more stealthy.

Due to the low request rate, this is not a rate-limiting vulnerability and as the attack comes from a single source; furthermore it is NOT a DDoS.

As an attacker, I can negatively impact the Fuel Wallets Browser Extension functionality by affecting the entire backend GraphQL server that is vulnerable to unexpected downtime (DoS) due to missing input validation on submitted queries.

\##IMPACT

As an attacker, by sending multiple resource-intensive queries, I can result in an impact that causes loss of functionality in the Fuel Wallet Browser Extension; Real-time information is no longer accessible along with any mutations that the GraphQL server is used for can no longer be executed effectively. The resulting impact targets 'taking down the application/website' and affects legitimate users from utilizing the legitimate Wallet Extension application functionality.

This not only affects the Browser Extension but it also indirectly affects the Fuel Network ecosystem due to the unexpected downtime negatively affecting user experiences, resulting in users loss of trust in the Fuel Network. This damages Fuel Networks reputation due to the low reliability of its services.

\##MITIGATION

In order to mitigate against this vulnerability, the developers need to implement

1. Restrictions on the the number of aliases that can be used. Unless, users are expected to use the GraphQL server, then alias batching should be disabled. (15,000 aliases is a large number)
2. A query cost analyzer should be used that prevents high cost queries from executing. This mitigation will prevent any other potential vectors not covered.
3. Furthermore, setting a suitable server timeout on queries will be an effective mitigation technique to prevent long-running queries.

## Proof of concept

\##PROOF-OF-CONCEPT

There are two PoCs provided - a Python3 script and a PoC video <https://youtu.be/ttjG5if2VaY> . Due to the first submission response, "delay for the single queries are not sufficient" the testing code in the Python3 script has the single queries commented out and execution jumps straight to the attack phase.

Testing single query phase (commented out): The Python3 script sends a single cyclic introspection query, and then follows up with the same query repeated using first 10 aliases, then 2000, 5000, 10000, 15000 and finally 17,000 aliases. Between each request there is a 10 second gap to avoid triggering Cloudflare WAF and to reduce any negative impact on the server.

\###Attack Phase (Executed after pressing case-insensitive 'y'):

The Python3 script implements an attack section that involves multiple queires being sent to the server with time delays of 10 seconds to in order to cause loss of functionality in the extension. The script at first runs with a low number of threads (5). For each attacking request sent, its corresponding response time is also shown in the console. The attack runs with low settings for minimal impact that does not disrupt the application for other users.

The provided video link - <https://youtu.be/ttjG5if2VaY> - shows the loss of functionality of the Fuel Wallet Browser Extension while the attack phase is running; specifically UI pages within the extension take longer to load, the extension claiming to have no assets/transaction history (due to failing to retrieve info) and 5xx errors being thrown within the extension UI.

\###Video PoC Description and Timestamps

* The video starts with highlighting the Python3 scripts settings (threads and time delay )
* The video shows the extension functionality working as normal when the attack is not running
  * 00:00:35 : The Attack starts
  * 00:01:34 : User action to triggering a GraphQL query to fetch the users transaction history. The UI window hangs for a long time before finally loading (different to previous loading time when attack was not running)
  * 00:02:00 : View the large response times being returned from the attack
  * 00:02:26 : User action to triggering a GraphQL query to fetch the users transaction history. The UI window hangs for a long time
  * 00:04:11 : The UI also hangs and eventually the wallet fails to retrieve assets and claims no assets are available.
  * 00:04:24 : First 5xx Server Error is thrown within the extension code
  * 00:04:34 : Fetching transaction history hangs for the longest time period yet. Another 5xx Server Error is thrown within the extension
  * 00:05:41 : Opening the wallet shows no assets are available and UI hangs
  * 00:06:10 : Opening the wallet again. The UI pages hangs for the longest time with no data shown.
  * 00:06:29 : Failed to Open the transaction history. No history is retrieved. Two different error messages are shown in the extension.
  * 00:06:57 : The attack is stopped
  * 00:07:06 : Showing normal functionality has resumed albeit a bit slower.

Steps to Reproduce Video

1. Have the Fuel Wallet Browser Extension installed in Chrome and be logged in to the wallet.
2. Use the Wallet as normal; loading the home page and triggering a Graphql query by visiting the Transaction History. Note that everything is working fine
3. Start the Python3 script in its attack phase. For every statement that says "Running", 5 threads have been sent to the server. After every 5 threads, there is a 10 second time delay in order to bypass the Cloudflare WAF
4. Let 2,3 more "Running" statements appear and try use the Browser extension. It should be running as normal
5. Wait for the first responses to be returned and try use the Browser extension. It should start to delay/ hang. If not, wait for a few more responses to be returned and try again.
6. Some useful indicators in loss of functionality, are a) if any assets are in the wallet, the extension says there are no assets. b) Trying to view the 'Transaction History' fails. The UI hangs for a long period of time. c) Some 'fetch' errors start to appear within the Browser Extension (this is because the extension queries the GraphQL server every second)
7. Once the indicators have shown themselves, stop the attack and wait for a few moments.
8. Try using the extension and notice that its functionality has been restored. If it still hangs, wait for a bit longer and try again.

Note: Some of the response times seem to be low, this is as some of them are receive 5xx errors but most hit the Graphql backend server.
