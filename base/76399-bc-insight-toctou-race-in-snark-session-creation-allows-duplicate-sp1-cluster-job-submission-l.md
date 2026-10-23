# 76399 bc insight toctou race in snark session creation allows duplicate sp1 cluster job submission leading to permanent proof loss

**Submitted on May 4th 2026 at 08:19:18 UTC by @Pig46940 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76399
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

`StatusPoller` and the `GetProof` RPC concurrently call `sync_and_update_proof_status()` in `base/crates/proof/zk/service/src/proof_request_manager.rs` on the same proof request with no mutual exclusion, allowing both to independently trigger SNARK aggregation when a STARK session completes. This creates two live Groth16 jobs on the SP1 cluster and two `SNARK` session rows in the database. In the worst-case ordering, where one SNARK session fails before the other completes, the proof request is permanently marked `Failed`, the valid receipt from the completing session is silently discarded, and the proof request has no automated recovery path despite a valid proof having been produced.

***

## Vulnerability Details

`OpSuccinctBackend::process_proof_request` determines whether to trigger stage-2 aggregation using a plain read from the database with no locking:

* `base/crates/proof/zk/service/src/backends/op_succinct/backend.rs`

```rust
// backend.rs line process_proof_request() 210-220
            let has_stark_completed = updated_sessions.iter().any(|s| {
                s.session_type == SessionType::Stark && s.status == DbSessionStatus::Completed
            });
            let has_snark_session =
                updated_sessions.iter().any(|s| s.session_type == SessionType::Snark);

            if has_stark_completed && !has_snark_session {
                info!(
                    proof_request_id = %proof_request.id,
                    "STARK completed, triggering stage-2 aggregation proof (SNARK Groth16)"
                );
```

Two concurrent callers can both read `has_snark_session = false` before either has committed a new session row. Both then enter `submit_aggregation_proof`, which calls `create_request()` on the SP1 cluster **before** writing to the database:

* `base/crates/proof/zk/service/src/backends/op_succinct/backend.rs`

```rust
// backend.rs  submit_aggregation_proof() line 687
        let cluster_proof_request = create_request(   // duplicate cluster job fires here
            artifact_client.clone(),
            ClusterElf::NewElf(AGGREGATION_ELF.to_vec()),
            stdin,
            &proof_config,
        )
        .await
        .map_err(|e| {
            error!(error = %e, "failed to submit aggregation proof to SP1 cluster");
            anyhow::anyhow!("Failed to submit aggregation proof to cluster: {e}")
        })?;
```

* `base/crates/proof/zk/service/src/backends/op_succinct/backend.rs`

```rust
// backend.rs  submit_aggregation_proof() line 720-727
        let session = CreateProofSession {
            proof_request_id: proof_request.id,
            session_type: SessionType::Snark,
            backend_session_id: cluster_proof_request.proof_id,
            metadata: Some(metadata),
        };

        repo.create_proof_session(session).await?;
```

The database provides no protection. `create_proof_session` in `repo.rs` is a plain `INSERT` with no `ON CONFLICT` clause, and the `proof_sessions` table has no `UNIQUE` constraint on `(proof_request_id, session_type)`:

* `base/crates/proof/zk/db/migrations/002_add_proof_sessions.sql`

```sql
-- 002_add_proof_sessions.sql
CREATE TABLE IF NOT EXISTS proof_sessions (
    id BIGSERIAL PRIMARY KEY,

    -- Reference to the proof request
    proof_request_id UUID NOT NULL REFERENCES proof_requests(id) ON DELETE CASCADE,

    -- Session metadata
    session_type VARCHAR(20) NOT NULL, -- 'STARK', 'SNARK', etc.
    backend_session_id VARCHAR(255) NOT NULL, -- Backend-specific session ID

    -- Status: RUNNING, COMPLETED, FAILED
    status VARCHAR(20) NOT NULL DEFAULT 'RUNNING',
    error_message TEXT,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE
);
```

The result is two live Groth16 jobs on the SP1 cluster and two `SNARK` rows in `proof_sessions` for the same proof request.

`determine_status` in `backend.rs` iterates sessions and immediately returns `Failed` upon encountering any session with `Failed` status, without evaluating the remaining sessions:

* `base/crates/proof/zk/service/src/backends/op_succinct/backend.rs`

```rust
// backend.rs determine_status() line 573-580
        for session in sessions {
            if session.status == DbSessionStatus::Failed {
                return ProofProcessingResult {
                    status: ProofStatus::Failed,
                    error_message: session.error_message.clone(),
                };
            }
        }
```

If either SNARK session fails while the other is still running, this loop marks the proof request terminal via `update_status_if_non_terminal` in `repo.rs`:

* `base/crates/proof/zk/db/src/repo.rs`

```sql
// repo.rs update_status_if_non_terminal() line 573-580
   pub async fn update_status_if_non_terminal(
        &self,
        id: Uuid,
        status: ProofStatus,
        error_message: Option<String>,
    ) -> Result<bool> {
        let result = sqlx::query(
            r#"
            UPDATE proof_requests
            SET
                status = $1,
                error_message = $2,
                completed_at = CASE WHEN $1 IN ('SUCCEEDED', 'FAILED') THEN NOW() ELSE completed_at END
            WHERE id = $3
              AND status NOT IN ('SUCCEEDED', 'FAILED')
            "#,
        )
        .bind(status.as_str())
        .bind(&error_message)
        .bind(id)
        .execute(&self.pool)
        .await?;

        let updated = result.rows_affected() > 0;

        Ok(updated)
    }
```

When the other session subsequently completes, `sync_session_with_sp1_cluster` in `backend.rs` calls `update_receipt_if_non_terminal`, which is gated on the same condition and becomes a no operation. The valid SNARK receipt is never written to the `proof_requests` row. The proof request is permanently stuck in `Failed` state with a valid SNARK receipt having been produced but never stored.

***

## Impact Details

This vulnerability matches the Medium impact category: a bug in the layer 2 network code that results in unintended behavior with no concrete funds at direct risk.

The race between `StatusPoller` and `GetProof` RPC unconditionally submits two Groth16 jobs to the SP1 cluster on every occurrence. If either SNARK session fails while the other is still running, `determine_status` marks the proof request permanently `Failed` while a valid proof is still being produced. When the second session subsequently completes, `update_receipt_if_non_terminal` is no operation, the valid SNARK receipt is silently discarded, and the proof request remains stuck in `Failed` state with no automated recovery path.

The trigger is a normal operational condition. `StatusPoller` runs on a fixed polling interval and `GetProof` RPC fires on every client query. Their overlap at STARK completion is not an adversarial edge case, it occurs in normal operation on every `OpSuccinctSp1ClusterSnarkGroth16` request where a client queries status around the STARK to SNARK transition.

No funds are directly at risk. The recovery cost, issuing a new `CreateProofRequest` and rerunning the full pipeline, is identical to that of any single SNARK session failure under normal operation.

### Potential Fix

The vulnerability has two layers that both require a fix.

**Layer 1: Database constraint**

Add a `UNIQUE(proof_request_id, session_type)` constraint to `proof_sessions` and change `create_proof_session` in `repo.rs` to use `INSERT ... ON CONFLICT DO NOTHING`. This prevents the second SNARK row from being created, but does not prevent the duplicate `create_request()` call to the SP1 cluster since that fires before the DB insert in `submit_aggregation_proof`.

```sql
ALTER TABLE proof_sessions
ADD CONSTRAINT unique_session_per_request_type
UNIQUE (proof_request_id, session_type);
```

* `base/crates/proof/zk/db/src/repo.rs`

```rust
// repo.rs  create_proof_session line 298
    pub async fn create_proof_session(&self, session: CreateProofSession) -> Result<i64> {
        let row = sqlx::query(
            r#"
            INSERT INTO proof_sessions (
                proof_request_id, session_type, backend_session_id, status, metadata
            )
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (proof_request_id, session_type) DO NOTHING
            RETURNING id
            "#,
        )
```

**Layer 2: Row-level lock before the check**

Wrap the `has_snark_session` check and `submit_aggregation_proof` call inside a serializable transaction with a `SELECT ... FOR UPDATE` on the `proof_requests` row. This ensures only one caller can pass the check and submit to the cluster.

Both fixes are needed together. Without the lock, the DB constraint stops duplicate rows but the SP1 cluster still receives two jobs. Without the DB constraint, the lock alone is sufficient but leaves the schema unprotected against future callers that bypass the lock path.

***

## References

* `base/crates/proof/zk/service/src/backends/op_succinct/backend.rs`
* `base/crates/proof/zk/db/src/repo.rs`
* `base/crates/proof/zk/db/migrations/002_add_proof_sessions.sql`
* `base/crates/proof/zk/service/src/proof_request_manager.rs`

## Proof of Concept

The following test can be added directly into the existing `mod tests` block in `proof_request_manager.rs`. It demonstrates that `StatusPoller` and `GetProof` RPC both pass the `has_snark_session` guard simultaneously and each create a separate SNARK session in the database.

`RacingSnarkBackend` mirrors the exact SNARK-trigger block from `OpSuccinctBackend::process_proof_request` in `backend.rs`: it calls `get_sessions_for_request`, evaluates `has_snark_session`, and calls `create_proof_session`. A `Barrier(2)` is inserted between the check and the insert, forcing both callers to complete the check before either proceeds to the insert. This makes the race window deterministic instead of relying on scheduling timing.

The precondition matches the exact production state that triggers the race: a `RUNNING` `OpSuccinctSp1ClusterSnarkGroth16` proof request with one `Completed` STARK session and no SNARK session. Both callers read `has_snark_session = false` simultaneously, both enter the branch, and both call `create_proof_session`. In production, `create_request()` on the SP1 cluster fires before `create_proof_session` inside `submit_aggregation_proof`, so two cluster jobs are unconditionally submitted before any DB write occurs.

The two assertions confirm both layers of the vulnerability. `snark_submissions == 2` proves both callers passed the guard. `snark_count == 2` proves the database has no constraint preventing two `SNARK` rows for the same proof request, confirming that `002_add_proof_sessions.sql` has no `UNIQUE(proof_request_id, session_type)` constraint and `create_proof_session` in `repo.rs` has no `ON CONFLICT` clause.

### PoC Code

```rust
use async_trait::async_trait;
use std::sync::atomic::{AtomicU32, Ordering};
use tokio::sync::Barrier;
use uuid::Uuid;

// Mirrors the SNARK-trigger TOCTOU in OpSuccinctBackend::process_proof_request.
// Barrier forces both callers past the `has_snark_session` check before either inserts,
// guaranteeing the race that in production relies on scheduling.
struct RacingSnarkBackend {
    gate: Arc<Barrier>,
    snark_submissions: Arc<AtomicU32>,
}

#[async_trait]
impl ProvingBackend for RacingSnarkBackend {
    fn backend_type(&self) -> BackendType { BackendType::OpSuccinct }
    fn name(&self) -> &'static str { "racing_snark" }
    async fn prove(&self, _: &base_zk_client::ProveBlockRequest) -> anyhow::Result<ProveResult> { unimplemented!() }
    async fn get_session_status(&self, _: &base_zk_db::ProofSession) -> anyhow::Result<SessionStatus> { unimplemented!() }

    async fn process_proof_request(
        &self,
        proof_request: &ProofRequest,
        repo: &ProofRequestRepo,
    ) -> anyhow::Result<ProofProcessingResult> {
        use base_zk_db::{CreateProofSession, SessionStatus as DbSessionStatus, SessionType};

        let sessions = repo.get_sessions_for_request(proof_request.id).await?;
        let has_stark_completed = sessions.iter().any(|s| {
            s.session_type == SessionType::Stark && s.status == DbSessionStatus::Completed
        });
        let has_snark_session = sessions.iter().any(|s| s.session_type == SessionType::Snark);

        // Both callers rendezvous here with has_snark_session=false, then both enter the branch.
        self.gate.wait().await;

        if has_stark_completed && !has_snark_session {
            self.snark_submissions.fetch_add(1, Ordering::SeqCst);
            // In production, create_request() fires on the SP1 cluster *before* this insert.
            repo.create_proof_session(CreateProofSession {
                proof_request_id: proof_request.id,
                session_type: SessionType::Snark,
                backend_session_id: format!("snark-{}", Uuid::new_v4()),
                metadata: None,
            }).await?;
        }
        Ok(ProofProcessingResult { status: ProofStatus::Running, error_message: None })
    }
}

#[sqlx::test(migrations = "../db/migrations")]
async fn test_concurrent_snark_trigger_creates_duplicate_sessions(pool: sqlx::PgPool) {
    use base_zk_db::{
        CreateProofRequest, CreateProofSession, SessionStatus as DbSessionStatus,
        SessionType, UpdateProofSession,
    };

    let repo = ProofRequestRepo::new(pool);

    // Precondition: RUNNING Groth16 request with a completed STARK session, no SNARK yet.
    let id = repo.create(CreateProofRequest {
        start_block_number: 100, number_of_blocks_to_prove: 1, sequence_window: None,
        proof_type: ProofType::OpSuccinctSp1ClusterSnarkGroth16, session_id: None,
        prover_address: Some("0x0000000000000000000000000000000000000001".to_string()), l1_head: None,
    }).await.unwrap();
    repo.update_status(id, ProofStatus::Running, None).await.unwrap();
    repo.create_proof_session(CreateProofSession {
        proof_request_id: id, session_type: SessionType::Stark,
        backend_session_id: "stark-001".to_string(), metadata: None,
    }).await.unwrap();
    repo.update_proof_session(UpdateProofSession {
        backend_session_id: "stark-001".to_string(),
        status: DbSessionStatus::Completed, error_message: None, metadata: None,
    }).await.unwrap();

    // Wire up manager.
    let snark_submissions = Arc::new(AtomicU32::new(0));
    let mut registry = BackendRegistry::new();
    registry.register(Arc::new(RacingSnarkBackend {
        gate: Arc::new(Barrier::new(2)),
        snark_submissions: Arc::clone(&snark_submissions),
    }) as Arc<dyn ProvingBackend>);
    let manager = ProofRequestManager::new(repo.clone(), Arc::new(registry));
    let pr = repo.get(id).await.unwrap().unwrap();

    // Simulate StatusPoller and GetProof RPC racing on the same proof request.
    let (m1, m2, pr1, pr2) = (manager.clone(), manager.clone(), pr.clone(), pr.clone());
    let (r1, r2) = tokio::join!(
        tokio::spawn(async move { m1.sync_and_update_proof_status(&pr1).await }),
        tokio::spawn(async move { m2.sync_and_update_proof_status(&pr2).await }),
    );
    r1.unwrap().unwrap();
    r2.unwrap().unwrap();

    // Both callers passed the guard → 2 cluster jobs submitted, 2 SNARK rows in DB.
    let snark_count = repo.get_sessions_for_request(id).await.unwrap()
        .into_iter().filter(|s| s.session_type == SessionType::Snark).count();
    assert_eq!(snark_submissions.load(Ordering::SeqCst), 2, "two cluster jobs submitted");
    assert_eq!(snark_count, 2, "two SNARK sessions in DB");
}
```

### Running PoC

Add the PoC code to the test module in `proof_request_manager.rs`, then run:

```bash
cd crates/proof/zk/service
cargo add --dev sqlx --features "postgres macros migrate runtime-tokio"
docker run --name azul-db -e POSTGRES_PASSWORD=password -p 5432:5432 -d postgres
cargo test -p base-zk-service --lib test_concurrent_snark_trigger_creates_duplicate_sessions -- --nocapture
```
