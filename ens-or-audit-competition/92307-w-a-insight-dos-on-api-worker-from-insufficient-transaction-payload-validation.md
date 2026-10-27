# 92307 w a insight dos on api worker from insufficient transaction payload validation

**Submitted on Sep 8th 2026 at 20:30:06 UTC by @blackgrease for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92307
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Taking down the application/website
  * Temporarily disabling user to access target site, such as:
* Locking up the victim from login
* Cookie bombing, etc.

## Description

The ENS API Worker is vulnerable to a DoS via Resource exhaustion disrupting the availability of the API and preventing legitimate users from carrying out their operations.

### Root cause

The authenticated `POST /transactions` endpoint accepts an attacker-controlled `payload` that is validated by `TransactionPayloadSchema` only as a generic record of string keys and arbitrary values without imposing any additional restrictions.

```ts
export const TransactionPayloadSchema = v.record(v.string(), v.unknown())
//Path: https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/shared-schema/src/transactions/index.ts#L30
```

The schema is applied to the request body through:

```ts
vValidator('json', UpsertTransactionSchema)
//Path: [audit-comp-ens/workers/api-worker/src/app/routes/transactions/index.ts](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/transactions/index.ts#L32)
```

After validation, the attacker-controlled payload is directly persisted to the database in:

```ts
payload: body.payload ?? null,
//Path: https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/transactions/index.ts#L47
```

Note that the `payload` field can also be written during the `onConflictDoUpdate` path:

```ts
//--snip--
: { operation: body.operation ?? null }),
...(body.name === undefined ? {} : { name: body.name ?? null }),
...(body.payload === undefined
 ? {}
: { payload: body.payload ?? null }),
//--snip--
//Path: https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/workers/api-worker/src/app/routes/transactions/index.ts#L59-L63
```

Additionally, the HTTP server has no configured request size limits, further enabling this attack vector.

As a result, an authenticated attacker can submit a large arbitrary HTTP request (e.g. \~30 MB) that passes validation and is processed and persisted by the application. This single large request when processed, causes a huge spike in both CPU and RAM usage on the server - both during and after - , subsequently disrupting the availability of the API server to other legitimate users as their requests cannot be executed.

### Downstream consequences

As the payload is consumed by different components/services in the API e.g Vite and Postgresql, the size of the malicious payload results in varying downstream consequences.

For example during testing:

1. With a single HTTP request of 30MB, the API server was unavailable to users for 20+ minutes. After the malicious request was processed and persisted, availability **did not resume** - authentication requests failed - making the DoS persistent and requiring manual intervention to resume availability.
2. Conversely, with an single HTTP request with a malicious payload of 10Mb, the API temporarily could not be reached by other clients. However, once the malicious payload was processed and persisted in the database, **availability resumed** and users could make requests again; making the DoS temporary.

In both cases, the attack resulted in PostgreSQL sustaining significantly elevated CPU and RAM consumption after processing the malicious payload.

The PoC video walks through the 30Mb instance as this was the most impactful exploit.

## Impact

* **Taking down the application/website:** As per the impacts definition, exploitation of this issue results in the disruption of the availability of the API application to legitimate users. The issue stems from a defect in the application's own logic - `TransactionPayloadSchema` - not imposing size or structural constraints on user controlled input. As shown in the PoC, during and after the attack, the API remains unavailabe and requests cannot be successfully made. For ENS users, this means they are unable to manage their account and handle any other Web based operations.
* **Temporarily disabling user to access target site:** As mentioned in the root cause, the size of the payload determines the overall down stream consequences. For smaller payloads - relative to the size that can result in complete disruption - once the attackers payload is processed, availability resumes making the impact a temporary disruption to users accessing the API application.

It is important to note that for servers with a large resource pool available, the attacker may submit multiple large requests to overload the service. (This does not need to be high volume) In the tested case, a single large HTTP request sufficed to make the Worker API unavailable in a low traffic environment.

Both impacts have been cited as both can be reached, depending on the size of the malicious payload albeit the PoC focuses on complete unavailability from a larger payload.

## Recommendations

1. Consider validating the payload size along with any string parameters.
2. Additionally, at the server level, enforce a size limit on the HTTP request.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/ca96517dc06c0ddc6f279016ba6a5f87>

## Proof of Concept

As proof of this issue, I have shared 20+ minute video of the attack - uploaded onto Google Drive. Additionally, the used script is shared in the private gist link.

The PoC video shows all users being able to access the API prior to the attack. During and after the attackers malicious request, legitmate user requests fail.

* PoC video on Google Drive: <https://drive.google.com/file/d/1qamqiV12YG6m70AEEsyP6KvVK2Gstpg7/view?usp=drive\\_link>
* Python3 script: <https://gist.github.com/blackgrease/ca96517dc06c0ddc6f279016ba6a5f87>

### The Environment

* The API Server: set up on a Cloud instance, running with 8Gb RAM.
* The attacker: set up on another Cloud instance
* Legitimate user1: set up on another Cloud instance and sends normal requests to the API server
* Legitimate user2: local and also sends normal requests to the API server
* The script:
  * Both attacker and client use the same script to reach the API server (though its been renamed depending on whos executing) however they use different payloads set via CLI args (`--payload-mb`) with the client using 1Kb and the attacker 30Mb. (In both cases, the script performs authentication prior and then sends the request)

### Video Timestamp Walkthrough

* 00:00 : Before the attackers malicious requests, all user (attackers and user) requests are responded successfully by the API (payload sent is 1kb in size).
* 00:33 : `htop` and `docker stats` showing the API's resources are stable with minimum consumption.
* 00:55 : **Attack starts.** Payload of 30Mb is sent to the API server
* 01:14 : **First large spike**: `htop` utility shows large spike in CPU and RAM consumption being associated with the API Vite process. A 1GB jump from its starting baseline
* 01:47 : Simulating a normal user trying to access the API server. The request is not answered throughout the attack.
* 03:30 : The API servers CPU/RAM usage remains high. Neither attacker or normal users request is responded to.
* 07:37 : Simulating another normal users request (Sent from local machine). Also gets no response from API server.
* 11:30 : Cancelled both the normal users requests and tried to send another request; both are unresponded to.
* 15:55 : **Second large spike**. `htop` shows RAM consumption exceeds 4GB and load shifts to postgresql database. `docker stats` shows an excess of 300% CPU Usage and increased RAM.
* 16:20 : Attackers single HTTP request is still not responded to and being processed by server.
* 17:00 : `docker stats` shows `api-worker-postgres-1` is in excess of 500% CPU usage
* 17:30 : Normal user availability remains down. Since attack has started no normal user requests have been responded
* 20:35: `htop` tracks RAM in excess of 5Gb and 100% CPU utilization. `docker stats` for the database remain in excess of 500%.
* 23:02 : Vite console shows the request finally being processed and displays the request body.
* 24:55: All requests - attackers and users return 500 error codes. Attacker gets a "Internal Server Error" even though request was processed on the backend. Users get a auth failure.
* 25:55 : Even though attack has stopped, user request still fail and server maintains high RAM/CPU usage across `htop` and `docker stats`
