# #39109 \[W\&A-Insight] syncStateDataGlobals will not work, effectively DoS'ing nodes

**Submitted on Jan 22nd 2025 at 12:40:06 UTC by @br0nz3p1ck4x3 for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39109
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * DOS

## Description

## Description

Inside `src/state-manager/ArchiverSyncTracker::syncStateDataGlobals()`, we find the following function that is being executed:

```typescript
          const getAccountDataByListFromArchiver = async (payload) => {
            const dataSourceArchiver = this.archiverDataSourceHelper.dataSourceArchiver
            const accountDataByListArchiverUrl = `http://${dataSourceArchiver.ip}:${dataSourceArchiver.port}/get_account_data_by_list_archiver`
            try {
              const result = await http.post(accountDataByListArchiverUrl, payload, false, 10000)
              console.log('getAccountDataByListFromArchiver result', result)
              return result
            } catch (error) {
              console.error('getAccountDataByListFromArchiver error', error)
              return null
            }
          }

          const result = await getAccountDataByListFromArchiver(signedMessage)
```

A `POST` call is made to the archiver. The payload of this call will be an array of `accountIds`.

The problem is, if the length of this array is more than 1, this call will return an error.

## Root Cause Analysis

The root cause is the following line in the archiver repo:[AccountDataProvider.ts#L269](https://github.com/shardeum/archiver/blob/0a38f31ecbeaca6be1d2092fa8b2b015ce4a7329/src/Data/AccountDataProvider.ts#L269)

```typescript
    const sql = `SELECT * FROM accounts WHERE accountId IN (?)`;
```

When using `(?)`, SQLite expects a single value, not an array. To properly handle an array of values with an IN clause, you need to create the correct number of parameter placeholders based on the array length.

## Impact

When the node is restoring and trying to sync, it will not be able to sync when fetching more than one `accountIds`, leading to an effective Denial of Service of the restoring node.

## Recommendation

Change the query such that it accommodates for multiple values inside the array. If you don't want to use multiple values, consider not using an array but a normal, constant value.

## Link to Proof of Concept

<https://gist.github.com/bronzepickaxe/fdbf068728a5262e8611624619370df1>

## Proof of Concept

## Proof of Concept

Check the gist here:

* <https://gist.github.com/bronzepickaxe/fdbf068728a5262e8611624619370df1>
