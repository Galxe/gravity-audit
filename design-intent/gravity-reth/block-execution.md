# Block Execution — Error & Panic Semantics

> How the block-execution pipeline treats errors and panics, and why `unwrap()` in certain paths is an accepted design choice rather than a latent DoS.

## Scope
- `crates/pipe-exec-layer-ext-v2/execute/src/lib.rs` — `filter_invalid_txs` (~line 1282), rayon-parallel transaction validity filtering, system-txn execution paths.
- Keywords: `filter_invalid_txs`, `rayon`, `par_iter`, `unwrap`, `panic`, `basic_ref`, `DBError`, `block-execution pipeline`, `DoS`.

## Design Intent

- **The block-execution pipeline is crash-stop on DB-level failure.** A state-DB read error (corrupted trie node, missing snapshot, transient disk error, pruned reference) is considered unrecoverable in-process: the block-execution pipeline cannot produce correct results without the underlying state, and continuing would risk silent consensus divergence.
- **`panic` and `Error` are treated as equivalent in this pipeline.** Whether the failure surfaces as a returned `Err` or a panic propagated from a rayon worker, the outcome is the same: the entire block-execution pipeline stops. There is no finer-grained recovery tier below this.
- **Node restart + DB repair is the intended recovery path**, not per-transaction error absorption.

## Invariants
- A `DBError` from the state DB is never transient-retriable from inside the execution pipeline — operator intervention (restart, resync, fix the DB) is required.
- Any logic that would "absorb" a DB error by demoting it to a per-tx rejection (e.g. treating `Err` as `Ok(None)` and marking the tx invalid) is prohibited, because it would allow a node whose DB is losing state to continue producing apparently-valid blocks that diverge from healthy nodes.

## Known Non-Issues

### `filter_invalid_txs` panics inside rayon parallel iterator on DB error — closed-as-intent, ref [#5](https://github.com/Galxe/gravity-audit/issues/5)
- **Surface finding.** `lib.rs:1282` calls `db.basic_ref(*sender).unwrap()` inside a `rayon` `into_par_iter` closure. `basic_ref` returns `Result<Option<AccountInfo>, DBError>`; `unwrap` on a `DBError` panics, and rayon propagates worker-thread panics to the caller, bringing down the entire block-execution pipeline. An adversary who can influence which sender addresses appear in the mempool could potentially steer the filter toward a pruned/unavailable state-trie node, turning it into a cheap DoS.
- **Why this is not a bug.** `panic` and `Err` are intentionally equivalent in this pipeline: both collapse block execution. Converting the `unwrap()` into a graceful `Err` branch that marks the sender's txs as invalid would be *strictly worse* — it would let a node with a broken DB silently produce divergent blocks. The explicit crash is the correct failure mode: loud, operator-visible, halts block production until the underlying DB issue is fixed. For a validator this means "halt until DB is repaired", which is the intended behaviour.
- **Boundary — when this would become a real bug.**
  - If an attacker can trigger the DB-error path *without* actually corrupting state (e.g. via a sender address that reliably causes `DBError` on a healthy node — a pure function-of-input DoS), then the panic becomes exploitable and must be addressed. Today the precondition is a real DB fault, not an attacker-controlled input.
  - If the pipeline ever gains a recovery tier that can safely continue execution on a partial DB, the unwrap must be replaced with explicit handling that plugs into that tier.
  - If `filter_invalid_txs` is ever reused outside the block-execution pipeline (e.g. in a mempool pre-filter where panic is unacceptable), the call-site must provide its own error handling.
- **Reference.** Issue [#5](https://github.com/Galxe/gravity-audit/issues/5), closing comment: _"reject: `panic` is the same as `Error`. Both will bring down the entire block-execution pipeline."_
