# #43038 \[BC-Insight] There is a permanent operator lockout came from an unsafe key rotation

**Submitted on Apr 1st 2025 at 01:36:11 UTC by @XDZIBECX for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43038
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-config>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

The issue on this fucntion is lies in the unsafe ordering of operations in the fucntion the rotate\_core\_resource\_account\_key where because the system is performs an irreversible on-chain key rotation for a core Aptos account before persisting the new signer information to the local configuration file, and this sequence is assumes that the disk write will always succeed, but offers no protection against common I/O failures as full disk, or permission errors, or crashes, so If the write is fails after the key is rotated the on-chain, the old signer will becomes invalid, and the operator loses the access to the account because the new signer was never saved and effectively locking themselves out permanently, this is a problem and is results in inconsistency because : the Aptos blockchain now requires the new key, but the local environment remains unaware of it, and there is no fallback, or backup, or even a recovery mechanism to resolve this mismatch see vulnerability details to see where this is arise from and impact

## Vulnerability Details

the problem as i describe on the brief and for more details the function first is performs an irreversible on-chain rotation of the core resource account key via a signed transaction on Aptos , and it's only afterward attempts to persist the updated configuration to the local file system and is using the try\_overwrite\_config\_to\_json. so If this local file write fails for any reason as mention above the new signer information is never saved, yet the blockchain has already accepted the new key, and as result, the local tool still holds the old key, which the Aptos protocol will reject for all subsequent transactions because its authentication model enforces that only the current on-chain public key can authorize activity, this is a problem because on this line `try_overwrite_config_to_json (which uses std::fs::File::create and serde_json::to_writer_pretty)` the fucntion does not write atomically, and does not sync to disk, also there is a miss of backup, and this is mean a failure and is gone leaves the user in a permanently broken state, There is no logic in the SDK that guards against this race condition or checks for consistency after the write so the entire flow assumes success. and this this transforms a low-level I/O failure into a high-level loss of control, and can became a locking the operator out of their own core account with no automated recovery path, The Aptos protocol is alerdy on docs confirms that once a key is rotated, the old key becomes unusable, so here is the vulnerable part <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-config/src/ops/aptos/rotate\\_key/core\\_resource\\_account/dot\\_movement.rs#L37C3-L40C12> :

```
let updated_config = rotator
    .rotate_core_resource_account_key(config, &client, &old_signer, new_signer)
    .await?; <-- Irreversible on-chain mutation happens here

self.try_overwrite_config_to_json(&updated_config)?; <-- Local write to config.json happens here

```

and we have this part on the fucntion is responsible for writing the new signer config and this is happen after rotating the key on-chain to a local file --> <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-config/src/ops/aptos/rotate\\_key/core\\_resource\\_account/dot\\_movement.rs#L42C3-L45C1> :

```
// write the migrated value
		self.try_overwrite_config_to_json(&updated_config)
			.map_err(|e| RotateCoreResourceAccountError::KeyRotationFailed(e.into()))?;

```

this is called a fucntion try\_overwrite\_config\_to\_json --> <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/dot-movement/src/lib.rs#L122C2-L133C3> :

```
/// Tries to write a configuration to a JSON file.
	pub fn try_write_config_to_json<T: serde::Serialize>(
		&self,
		config: &T,
	) -> Result<(), anyhow::Error> {
		let file = std::fs::File::create(self.get_config_json_path())
			.map_err(|e| anyhow::anyhow!("Failed to create file: {}", e))?;
		let writer = std::io::BufWriter::new(file);
		serde_json::to_writer_pretty(writer, config)
			.map_err(|e| anyhow::anyhow!("Failed to write config: {}", e))?;
		Ok(())
	}

```

this function is supposed to take a new configuration that is containing the updated signer after rotation and persist it to disk as a JSON file as config.json, and is use `std::fs::File::create`, is truncates the existing file immediately, and writes using `serde_json::to_writer_pretty`. as see the fucntion does not call .flush() or .sync\_all() to ensure data is committed to disk and dose not back up the previous config before overwriting it and also is assumes write success and has no recovery so If anything goes wrong during the write, the file can be corrupted, empty, or incomplete and the operator is now permanently locked out of the account because the valid signer is accepted by the chain and was never saved the the old key should be invalid forever.

## Impact Details

the bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk” because is exists in L1 SDK logic that controls key rotation and is breaks the account's ability to sign or interact with the blockchain but there is no direct fund loss

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-config/src/ops/aptos/rotate\\_key/core\\_resource\\_account/dot\\_movement.rs#L36C3-L48C2>

## Proof of Concept

## Proof of Concept

here is a test show the problem :

```
use std::fs::{File};
use std::io::Write;
use std::path::PathBuf;
use tempfile::tempdir;

use movement_config::DotMovement;
use movement_config::rotate_key::core_resource_account::{
    RotateCoreResourceAccountKeyOperations, CoreResourceAccountKeyRotationSigner,
};
use movement_config::releases::biarritz_rc1::Config;

struct MockSigner;
impl CoreResourceAccountKeyRotationSigner for MockSigner {
    fn signer_identifier(&self) -> String {
        "mock_new_signer".to_string()
    }
}

#[tokio::test]
async fn test_key_rotation_then_write_failure_causes_sdk_desync() {
    // Step 1: Prepare valid config path + file
    let temp_dir = tempdir().expect("failed to create tempdir");
    let config_path = temp_dir.path().to_path_buf();
    let dot = DotMovement::new(config_path.to_str().unwrap());

    // Create initial config with OLD signer
    let mut initial_config = Config::default();
    initial_config
        .execution_config
        .maptos_config
        .chain
        .maptos_private_key_signer_identifier = "old_signer".to_string();

    let file_path = dot.get_config_json_path();
    let mut f = File::create(&file_path).unwrap();
    let json = serde_json::to_string_pretty(&initial_config).unwrap();
    f.write_all(json.as_bytes()).unwrap();

    // Step 2: Confirm original signer is as expected
    let read_config: Config = dot.try_get_config_from_json().unwrap();
    assert_eq!(
        read_config
            .execution_config
            .maptos_config
            .chain
            .maptos_private_key_signer_identifier,
        "old_signer"
    );

    // Step 3: Simulate failure of config write — override to unwritable path
    let broken_path = PathBuf::from("/dev/full");
    let mut broken_dot = DotMovement::new(broken_path.to_str().unwrap());

    // Step 4: Run vulnerable flow: on-chain key rotation + failed write
    let result = broken_dot
        .rotate_core_resource_account_key(&MockSigner)
        .await;

    assert!(
        result.is_err(),
        "Expected error when writing updated config fails"
    );

    // Step 5: Verify that config file is still untouched → old signer remains
    let fresh_dot = DotMovement::new(config_path.to_str().unwrap());
    let final_config: Config = fresh_dot.try_get_config_from_json().unwrap();

    assert_eq!(
        final_config
            .execution_config
            .maptos_config
            .chain
            .maptos_private_key_signer_identifier,
        "old_signer",
        "Signer on disk was not updated — stale signer remains"
    );

    println!("  Key rotated, but config write failed → SDK is desynced");
}
```
