# gravity-sdk

> Consensus layer driving gravity-reth. Orders transactions into blocks, runs BFT to finalise them, and observes cross-chain / DKG / JWK events that reth must incorporate into each block.

## Overview

[gravity-sdk](https://github.com/Galxe/gravity-sdk) is a modular consensus layer based on Aptos-BFT (pipelined BFT: ordering and execution-finality are separate phases). For Gravity Chain it plays the role of "consensus engine": the SDK decides ordering and finality, gravity-reth decides state transitions. The boundary between the two is the **pipe exec layer** (`crates/pipe-exec-layer-ext-v2`) in gravity-reth.

Entry materials worth reading on first pass:
- `gravity-sdk/readme.md`
- `gravity-sdk/Gravity_SDK_Consensus_Bridge_Flow.md`

## How gravity-reth depends on it

- Direct Cargo dependency: `gravity-api-types` (git, currently pinned at rev `16ed314689713d84995ac484210630292d6633c7`) — the shared wire types for the bridge.
- Bridge implementation lives entirely in gravity-reth: **`crates/pipe-exec-layer-ext-v2/`**
  - `execute/` — `PipeExecLayerApi`, block execution, system-txn construction, precompiles.
  - `relayer/` — oracle relayer consumed by the SDK side (`OracleRelayerManager`, `BlockchainEventSource`).
  - `event-bus/` — internal signalling between execution and the SDK-facing surface.

## Interaction Surface

### 1. Ordered block push — SDK → reth
- **API**: `PipeExecLayerApi::push_ordered_block(OrderedBlock)` (and underlying unbounded MPSC channel `ordered_block_tx`)
- **Data**: `OrderedBlock { epoch, id, parent_id, number, transactions, extra_data: Vec<ExtraDataType>, randomness: U256 }`
- **Purpose**: hand a fully-ordered block to reth for execution.
- **reth side**: `crates/pipe-exec-layer-ext-v2/execute/src/lib.rs` (receive loop around `1336-1337`).

### 2. Execution result pull — reth → SDK
- **API**: `PipeExecLayerApi::pull_executed_block_hash()` (async, backed by MPSC `execution_result_rx`).
- **Data**: `ExecutionResult { block_id, block_hash, block_number, txs_info: Vec<TxInfo>, gravity_events: Vec<GravityEvent> }`
- **Purpose**: return reth's computed block hash + per-tx info + emitted Gravity events so the SDK can run BFT voting over the hash.
- **reth side**: `lib.rs:1347-1348`.

### 3. Verified-hash commit — SDK → reth
- **API**: `PipeExecLayerApi::commit_executed_block_hash(block_id, Option<block_hash>)`; synchronised via a custom `Channel<K,V>` (`verified_block_hash_rx`).
- **Purpose**: once ≥ 2/3 of validators sign the same hash, SDK tells reth the block is final and can be persisted.
- **reth side**: `lib.rs:1355-1360`.

### 4. Validator / Extra-data payloads — SDK → reth (inside `OrderedBlock`)
- **Variants**: `ExtraDataType::Metadata`, `ExtraDataType::DKG(Vec<u8>)`, `ExtraDataType::JWK(Vec<u8>)`.
- **Purpose**: carry DKG transcripts, JWK oracle updates, and block metadata into the block body as system transactions.
- **reth side**: converted to system txns by `construct_validator_txn_from_extra_data()` in `crates/pipe-exec-layer-ext-v2/execute/src/onchain_config/mod.rs:142`. Sorting of the vector (DKG before JWK) happens at `lib.rs:752-756` prior to execution.

### 5. Persistence notification — reth → SDK
- **API**: `PipeExecLayerApi::wait_for_block_persistence(block_number)` (async; oneshot reply routed through `PipeExecLayerEventBus` via `WaitForPersistenceEvent { block_number, tx }`).
- **Purpose**: SDK waits for reth to confirm the finalised block is durably stored before ACKing up the consensus pipeline.
- **reth side**: `lib.rs:1365-1373`.

### 6. Oracle / Relayer — reth feeds SDK
- **Component**: `OracleRelayerManager` + `BlockchainEventSource` (in gravity-reth).
- **Purpose**: poll external chains (e.g. Ethereum `MessageSent(nonce, blockNumber, payload)` from the Gravity portal), surface `PollResult { jwk_structs, max_block_number, nonce, updated }` for the SDK's JWK/oracle observer, which then injects them back into a future block via `ExtraDataType::JWK`.
- **reth side**: `crates/pipe-exec-layer-ext-v2/relayer/src/oracle_manager.rs:44-150`.
- **SDK side** (conceptual): registers `GLOBAL_RELAYER` and pulls `PollResult`s via `gaptos::jwk_observer`.

## Invariants gravity-reth assumes gravity-sdk upholds

1. **Block-id stability.** Once SDK pushes a block with `id = X`, any future `commit_executed_block_hash(id, …)` for that block uses the same `X`. Mismatches are hard-asserted on the reth side (`lib.rs:420`).
2. **Epoch monotonicity.** Incoming blocks never decrease in epoch. Blocks whose `epoch < current_epoch` are silently discarded as idempotent late arrivals (`lib.rs:398-407`).
3. **Sequential numbering within an epoch.** Block N for epoch E only arrives after reth has finished executing block N-1 of the same epoch (enforced by `execute_block_barrier: Channel<(epoch, block_number), _>` with a 2s timeout, `lib.rs:386-418`).
4. **Well-formed `extra_data`.** Each `ExtraDataType` entry decodes correctly; no partial or truncated payloads. DKG entries come before JWK entries once sorted (the sort itself is done by reth; SDK must not send duplicates under a given semantic nonce).
5. **Unique DKG finish per epoch.** SDK ensures `finishTransition` is submitted at most once per epoch (no double-submission while `DKG.hasInProgress`).
6. **Serial per-URI oracle polling.** SDK's relayer observer calls `poll_uri` for a given URI strictly serially (no concurrent calls on the same URI). This is load-bearing for the relayer's Relaxed-atomic design (see [../../design-intent/gravity-reth/relayer-oracle.md](../../design-intent/gravity-reth/relayer-oracle.md)).
7. **At-least-once delivery is acceptable.** SDK/consensus is the nonce-dedup authority; reth does not need to guarantee exactly-once forwarding from the relayer.

## Invariants gravity-sdk assumes gravity-reth upholds

1. **Deterministic execution.** The same `OrderedBlock` always produces the same `block_hash` and the same receipt-log sequence across every honest node. Any non-determinism breaks BFT finality.
2. **System-txn precedence.** Within a block, the `onBlockStart` metadata txn executes first; then the `extra_data`-derived validator txns (DKG before JWK); only then user transactions. Execution of these is walked through `lib.rs:700-827`.
3. **Epoch-change atomicity.** When a DKG system txn emits `NewEpochEvent`, reth short-circuits further txn execution for the block and returns `EpochChanged(…)` (`lib.rs:795-814`). See [../../design-intent/gravity-reth/consensus-epoch.md](../../design-intent/gravity-reth/consensus-epoch.md) for why this is by design.
4. **Verified-hash reception is durable.** After `commit_executed_block_hash` is accepted, reth persists and fires the `WaitForPersistenceEvent` reply. Losing this = losing the finality proof.
5. **Precompile / system-contract addresses exist & behave.** `JWK_MANAGER_ADDR`, `NATIVE_MINT_PRECOMPILE_ADDR`, `RECONFIGURATION_WITH_DKG_ADDR`, `SYSTEM_CALLER`, etc. (`onchain_config/mod.rs:77-78`) are always deployed and callable.

## Audit hot spots

1. **Epoch transition races.** The `epoch` atomic (`lib.rs:352`) and the `block_epoch` arrival check (`lib.rs:398`) are not a single atomic read-and-check. A 2-second barrier timeout could, in principle, drop a valid block if an epoch race occurs during the wait. Confirm that `BlockBufferManager` on the SDK side serialises epoch N commits before epoch N+1 starts.
2. **Malformed `extra_data`.** Failure to decode a JWK/DKG payload in `construct_validator_txn_from_extra_data` currently logs + skips (`lib.rs:766-774`); there is no retry or SDK-side notification. A reviewer should check whether silent drop of cross-chain events is safe (today it relies on the event being re-emitted by the source chain and re-polled).
3. **Hash-mismatch deadlock.** If `commit_executed_block_hash(id, Some(H))` arrives while reth has already persisted hash H′ ≠ H, the defensive check never fires the notify (`lib.rs:1355-1360`). Understand who guarantees H equals reth's computed hash before commit.
4. **Channel closure / starvation.** `ordered_block_tx` / `execution_result_rx` have no health monitor; an unexpected close-on-one-side deadlocks the other. Confirm the SDK holds the other half for the node's lifetime.
5. **Relayer nonce replay across orphaned blocks.** A cross-chain event `n=N` included in an orphaned reconfiguration block may be re-injected in block N+2 after the orphan. The NativeOracle nonce check must reject the replay; see the boundary notes in [../../design-intent/gravity-reth/relayer-oracle.md](../../design-intent/gravity-reth/relayer-oracle.md).
6. **Validator-tx ordering within `extra_data`.** The sort at `lib.rs:752-756` enforces DKG-before-JWK. A reviewer should verify nothing downstream reintroduces JWK-before-DKG execution (e.g. via concurrent construction).
