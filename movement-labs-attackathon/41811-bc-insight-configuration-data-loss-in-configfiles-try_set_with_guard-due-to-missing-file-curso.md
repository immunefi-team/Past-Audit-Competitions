# #41811 \[BC-Insight] Configuration data loss in configfile's \`try\_set\_with\_guard\` due to missing file cursor reset

**Submitted on Mar 18th 2025 at 15:50:19 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41811
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/util/godfig>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

ConfigFile struct’s `try_set_with_guard` function doesnt reset the file cursor before reading making the entire configuration file to be overwritten with each new key-value pair set operation. This results in the loss of all prior configuration data, which in a production environment couldcause misconfigurations.

## Vulnerability Details

Take a look at the `try_set_with_guard` function of the ConfigFile struct, which is meant to update the JSON content of a file with a new key-value pair. The function is designed to:

1.) Read the current contents of the file. 2.) Parse it as JSON.\
3.) Modify the JSON based on the provided key and value. 4.)Write the updated JSON back to the file.

But here's the issue; the function does not seek to the start of the file before reading its contents. This leads to incorrect cases because the file’s cursor position is not reset, causing the function to read from the wrong position—typically the end of the file—resulting in an empty string being read instead of the actual file contents.

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/godfig/src/backend/config\\_file/mod.rs#L70-L77>

```rust
let mut contents = String::new();
		// write_guard.seek(std::io::SeekFrom::Start(0)).await?;
		write_guard.read_to_string(&mut contents).await?;
		let mut json: serde_json::Value = if contents.is_empty() {
			serde_json::Value::Object(serde_json::Map::new())
		} else {
			serde_json::from_str(&contents)? // parse the contents as JSON (if any
		};
```

The seek call is commented out, meaning the file cursor remains at its current position when `read_to_string` is called.

In contrast, the `try_get_with_guard` function correctly seeks to the start before reading:

```rust
write_guard.seek(std::io::SeekFrom::Start(0)).await?;
write_guard.read_to_string(&mut contents).await?;
```

Since `ConfigFile` uses a shared file handle wrapped in an `Arc<FileRwLock<File>>`, the file cursor’s position persists across operations. After a `try_get` or a previous `try_set`, the cursor is likely at the end of the file. When `try_set_with_guard` attempts to read without seeking to the start, it reads from the end, resulting in `contents` being an empty string.\
When `contents` is empty, the function initializes `json` as a new, empty JSON object (`serde_json::Value::Object(serde_json::Map::new())`), modifies it with the new key-value pair, and writes it back. This effectively overwrites the entire file with only the new key-value pair, discarding any previous data.

## Impact Details

This causes the `ConfigFile` to lose all existing key-value pairs each time a new value is set, instead of updating the existing JSON structure. For example:

1. Initial State: The file contains `{"key1": 42}`.
2. Set Operation: Call `try_set(vec!["key2"], Some(43))`.

* Without seeking to the start, `read_to_string` reads an empty string.
* A new JSON object is created, and `{"key2": 43}` is set.
* The file is overwritten with `{"key2": 43}`, erasing `{"key1": 42}`.

3. Result: Getting `["key1"]` returns `None`, even though it was previously set.

This doesnt align with the expected functionality of a config file backend, where multiple keys should coexist and persist across operations. The provided tests (e.g., `test_get_set`, `test_transaction` ) don’t catch this because they only operate on a single key, masking the issue.

The [`Manager`](https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/manager.rs#L10-L16) in the light-node module uses `Godfig` with `ConfigFile` as the backend. When it updates any configuration parameter, it would inadvertently delete all other parameters, causing the light node to operate with incomplete configuration.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/godfig/src/backend/config\\_file/mod.rs#L70-L77>

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/manager.rs#L10-L16>

## Proof of Concept

## Proof of concept

```rust
#[tokio::test]
async fn test_multiple_keys() -> Result<(), anyhow::Error> {
    let file = tempfile::tempfile()?;
    let config_file = ConfigFile::new(file.into());

    // Set first key
    config_file.try_set(vec!["key1".to_string()], Some(42)).await?;
    
    // Verify first key was set
    let result1_before = config_file.try_get::<_, i32>(vec!["key1".to_string()]).await?;
    assert_eq!(result1_before, Some(42), "First key should be set correctly");
    
    // Set second key
    config_file.try_set(vec!["key2".to_string()], Some(43)).await?;

    // Get both keys
    let result1_after = config_file.try_get::<_, i32>(vec!["key1".to_string()]).await?;
    let result2 = config_file.try_get::<_, i32>(vec!["key2".to_string()]).await?;

    // This assertion fails because key1 is lost after setting key2
    assert_eq!(result1_after, Some(42), "First key should still exist after setting second key");
    assert_eq!(result2, Some(43), "Second key should be set correctly");
    
    Ok(())
}
```

Test fails because after setting `key2`, the file only contains `{"key2": 43}`, and `key1` is lost.

## Fix

Uncomment or add the line `write_guard.seek(std::io::SeekFrom::Start(0)).await?;` before reading the file contents in the `try_set_with_guard` function.
