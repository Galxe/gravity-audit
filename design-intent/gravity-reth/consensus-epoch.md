# Consensus — Epoch Transition & Block Body

> How gravity-reth composes the block body and state bundle during a DKG-triggered epoch change, and why the seeming mismatch is intentional.

## Scope
- `crates/pipe-exec-layer-ext-v2/execute/src/lib.rs` — `execute_system_transactions`, `into_executed_ordered_block_result` (around lines 700–830).
- `crates/pipe-exec-layer-ext-v2/execute/src/onchain_config/metadata_txn.rs` — block body assembly.
- `crates/pipe-exec-layer-ext-v2/execute/src/onchain_config/dkg.rs` — `finishTransition` system transaction.
- Keywords: `epoch change`, `NewEpochEvent`, `take_bundle`, `DKG-triggered`, `reconfiguration`, `block body`, `state bundle`, `metadata txn`, `onBlockStart`.

## Design Intent

- **NIL-block epoch boundary.** When the Reconfiguration contract signals that the current block should end the epoch, gravity-reth does NOT pack the triggering DKG system transaction, the `onBlockStart` metadata txn, and other prior system txns into the same block as a normal EVM block. The epoch-change block is effectively a **reconfiguration boundary block**: its on-chain body carries only the system transaction(s) that caused the emission, while the state bundle captures the full reconfiguration delta.
- **Consensus layer is the authority on block validity at epoch boundaries.** Finality/verification at the epoch boundary flows through `gravity-sdk` (which observes `NewEpochEvent` and knows the block is a reconfiguration block) — not through independent EVM-style re-execution by third parties. Generic EVM consumers are not expected to re-execute reconfiguration blocks against the parent state and match the state root.
- **One new-epoch system tx per epoch-changing block.** A new-epoch system transaction is emitted only when a `NewEpochEvent` actually fires. The executor never speculatively inserts a new-epoch txn and then rolls it back.

## Invariants
- A block produces a `NewEpochEvent` at most once; when it does, the block is treated as an epoch-boundary block end-to-end (header, body, bundle, consensus-layer handling).
- All prior system transactions in the block (`onBlockStart` + any earlier validator txns) are already part of the state bundle by the time the DKG txn emits `NewEpochEvent`.
- The consensus layer (`gravity-sdk`) decides finality with full knowledge of which block is a reconfiguration block; see [../../dependency/gravity-reth/gravity-sdk.md](../../dependency/gravity-reth/gravity-sdk.md).

## Known Non-Issues

### DKG-triggered epoch change produces block body inconsistent with state bundle — closed-as-intent, ref [#2](https://github.com/Galxe/gravity-audit/issues/2)
- **Surface finding.** At `lib.rs:795-816`, when a DKG validator txn triggers an epoch change, `executor.take_bundle()` returns accumulated state from the metadata `onBlockStart` txn plus every prior validator txn plus the triggering DKG txn, but `into_executed_ordered_block_result` constructs the block body with only the triggering txn (`metadata_txn.rs:97`: `body: BlockBody { transactions: vec![self.txn], .. }`). A third party re-executing the block from its transaction list against the parent state will compute a different state root than the committed bundle, ostensibly breaking a core EVM-block invariant.
- **Why this is not a bug.** A new-epoch system transaction is only ever emitted when `new-epoch` actually fires, and when it fires the block is a reconfiguration boundary, not a normal EVM block. Generic EVM-style re-execution consumers are not in scope for this block type; verification is done via the consensus layer, which sees the reconfiguration transcript and the full validator-set transition explicitly. The state bundle is authoritative; the single-txn body carries the cause of the reconfiguration for archival and on-chain readability.
- **Boundary — when this would become a real bug.**
  - If a non-boundary block were to start carrying this asymmetric (body ⊂ bundle) shape, that is a bug.
  - If any component downstream of the execution layer (state sync, archive node, fraud prover, light client, indexer) is introduced that is expected to re-derive the state root from the stored body alone, this design must be revisited.
  - If an epoch-changing block ever contains a non-system user transaction mixed in with the reconfiguration, the body/bundle skew stops being explainable.
- **Reference.** Issue [#2](https://github.com/Galxe/gravity-audit/issues/2), closing comment: _"reject: There's only new-epoch system transaction if new-epoch emitted."_
