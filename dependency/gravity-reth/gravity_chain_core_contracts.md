# gravity_chain_core_contracts

> Solidity system contracts that gravity-reth drives via system transactions (`SYSTEM_CALLER`) and reads via `eth_call`. They encode staking, validator membership, epoch transitions, DKG, timestamp, and cross-chain / JWK oracle state.

## Overview

`gravity_chain_core_contracts` is a layered Solidity contract suite (runtime / staking / blocker / oracle layers). Every block, gravity-reth injects one or more **system transactions** sent from the well-known `SYSTEM_CALLER` address; and periodically it reads contract state via static calls to build consensus-layer data structures (validator set, DKG config, performance, etc.). The consensus layer (`gravity-sdk`) drives this cycle through `ExtraDataType::{Metadata,DKG,JWK}` entries inside each `OrderedBlock`.

Entry materials worth reading on first pass:
- `gravity_chain_core_contracts/README.md`, `ABOUT.md`
- `gravity_chain_core_contracts/DKG_PROCESS.md`
- `gravity_chain_core_contracts/randomness.md`
- `gravity_chain_core_contracts/spec_v2/`

## How gravity-reth depends on it

- ABIs are surfaced in reth via `alloy_sol_types` `sol!` bindings, primarily under:
  - `crates/pipe-exec-layer-ext-v2/execute/src/onchain_config/` — one module per contract family:
    - `mod.rs` — `construct_validator_txn_from_extra_data`, `new_system_call_txn`, `transact_system_txn`, system-contract address constants.
    - `metadata_txn.rs` — Blocker / Timestamp / Reconfiguration block-start txn.
    - `dkg.rs` — DKG state read, `finishTransition` construction.
    - `jwk_oracle.rs` — NativeOracle `record` / `recordBatch` construction.
    - `validator_set.rs` — `ValidatorManagement` reads.
    - `performance.rs` — `ValidatorPerformanceTracker` reads.
    - `types.rs` — shared type conversions (e.g. `wei_to_ether`).

## System contracts at a glance

| Contract | Address (conceptual) | Role |
|----------|----------------------|------|
| `Timestamp` | `0x…1625f1000` | Microsecond-precision chain clock, advanced by Blocker each block. |
| `EpochConfig` | `0x…1625f1005` | Epoch interval config; read by Reconfiguration to decide transitions. |
| `RandomnessConfig` | `0x…1625f1003` | DKG thresholds (secrecy / reconstruction / fast-path). |
| `DKG` | `0x…1625f2002` | DKG session lifecycle + transcript storage. |
| `ValidatorManagement` | `0x…1625f2001` | Active / pending-active / pending-inactive validator sets. |
| `Reconfiguration` | `0x…1625f2003` | Epoch orchestrator (`checkAndStartTransition`, `finishTransition`). |
| `Blocker` | `0x…1625f2004` | Block prologue (`onBlockStart`). |
| `ValidatorPerformanceTracker` | `0x…1625f2005` | Proposal success/failure accounting. |
| `NativeOracle` | `0x…1625f4000` | Records verified cross-chain / JWK data (nonce-sequenced). |
| `JWKManager` | `0x…1625f4001` | OAuth provider JWK store; NativeOracle callback target. |
| `SYSTEM_CALLER` | `0x…1625f0000` | The address reth signs all system txns with. |

## Interaction Surface

### Per-block prologue — reth → Blocker/Timestamp/Reconfiguration
- **reth side**: `metadata_txn.rs:224-241` (`construct_metadata_txn`).
- **Direction**: WRITE via system tx (`Blocker.onBlockStart(proposerIndex, failedProposerIndices, timestampMicros)`).
- **Purpose**: advance chain clock, record proposer / failed proposers, trigger epoch transition check (`Reconfiguration.checkAndStartTransition()` is invoked inside `onBlockStart`).
- **NIL-block convention**: when there is no proposer, reth passes `proposerIndex = u64::MAX`; Timestamp interprets this as SYSTEM_CALLER and allows time to remain unchanged.

### Epoch transition — reth ↔ Reconfiguration/DKG
- **Read** (reth → DKG): `dkg.rs:121-145` via `eth_call getDKGState()` returns `(lastCompleted, hasLastCompleted, inProgress, hasInProgress)`.
- **Read** (reth → RandomnessConfig): `dkg.rs:173-189` via `eth_call` for current/pending DKG thresholds.
- **Read** (reth → ValidatorManagement): `validator_set.rs:38-98` — `getActiveValidators`, `getPendingActiveValidators`, `getPendingInactiveValidators`.
- **Write** (reth → Reconfiguration via DKG): `dkg.rs:374-412` constructs `finishTransition(transcript)` once the SDK-side DKG protocol yields a transcript; on-chain this applies all pending configs, increments `currentEpoch`, emits `NewEpochEvent`.
- Full protocol narrative: `DKG_PROCESS.md`.

### JWK / cross-chain oracle — reth → NativeOracle
- **reth side**: `jwk_oracle.rs:230, 319` (`construct_oracle_record_transaction` → `NativeOracle.record(sourceType, sourceId, nonce, blockNumber, payload, callbackGasLimit)` or `recordBatch`).
- **Direction**: WRITE via system tx.
- **Data shape**: `sourceType ∈ {0 blockchain, 1 JWK}`, `sourceId` = provider/chain id, `nonce` strictly increasing per `(sourceType, sourceId)`, `payload` = ABI-encoded data, optional callback gas.
- **Callback**: NativeOracle invokes `JWKManager.onOracleEvent()` (or other registered handler); a revert in the callback does **not** revert the `record()` call itself.

### Validator performance — reth ← ValidatorPerformanceTracker
- **reth side**: `performance.rs` — `eth_call` for `IndividualPerformance[]` (successful / failed proposal counts). Used during epoch transitions for auto-eviction decisions.

### System-transaction construction primitives
- **`new_system_call_txn`** at `onchain_config/mod.rs:177-195`: builds a legacy `TransactionSigned` with `gas_limit: 30_000_000`, zero value, zero signature; sender is recovered as `SYSTEM_CALLER` via `Recovered::new_unchecked`.
- **`transact_system_txn`** at `metadata_txn.rs:185-212`: executes the system tx via revm, logs failures gracefully (no hard assert).
- Nonces are sequential within a block: metadata at position 0, subsequent validator txns at positions 1+.

## Invariants gravity-reth assumes the contracts uphold

1. **Return shape fidelity.** `getDKGState`, `getActiveValidators`, `currentEpoch()`, `getCurrentConfig`, etc. return exactly the tuple shape declared in the `sol!` bindings. Schema drift in the contracts without coordinated reth update → decode panic.
2. **Event emission.** `DKGStartEvent`, `NewEpochEvent`, `BlockStarted` are emitted on their defined triggers. Consensus and reth both depend on these.
3. **Access control is enforced on-chain.** Every sensitive entry point (e.g. `finishTransition`, `Blocker.onBlockStart`, oracle `record`) reverts if `msg.sender != SYSTEM_CALLER` (for system-only endpoints). reth does not re-check; it trusts contract-side `requireAllowed(SYSTEM_CALLER)`.
4. **Single in-flight DKG.** `DKG.hasInProgress = true` prevents a new `start()` while one is open; `finishTransition` clears it.
5. **Oracle nonce strictly increasing per `(sourceType, sourceId)`**, starts at 1, no backfill.
6. **Time monotonicity.** Normal blocks: `newTime > currentTime`. NIL blocks: `newTime == currentTime`. Violations cause `updateGlobalTime` to revert.
7. **Pending-config semantics.** `RandomnessConfig / ConsensusConfig / ExecutionConfig / ValidatorConfig` apply pending changes only at epoch boundary via `applyPendingConfig`; reth must re-read after epoch transition.
8. **Immutable deployment.** System contracts are singletons at known addresses and are not behind upgradeable proxies.

## Invariants the contracts assume gravity-reth upholds

1. **`SYSTEM_CALLER` identity.** Only reth constructs txns from `SYSTEM_CALLER`; `Recovered::new_unchecked(txn, SYSTEM_CALLER)` is applied to every system txn before execution.
2. **Nonce sequencing within a block.** System txns run in order: metadata first, then validator txns (DKG before JWK), then user txns.
3. **Microsecond timestamps.** `Timestamp.updateGlobalTime` receives µs (u64), not seconds. reth reads `orderedBlock.timestamp_us`.
4. **NIL-block signalling.** `proposerIndex = u64::MAX` signals a NIL block; Blocker expects this sentinel rather than a random address.
5. **Epoch determinism.** All nodes detect epoch elapsed at the same block height, because they read `EpochConfig.epochIntervalMicros` from the same state.
6. **No partial commits.** Every system txn either fully succeeds or reverts; reth does not commit partial state from a failed system txn.
7. **Submit `finishTransition` exactly once per DKG session.** Double-submit is a reth-side contract violation (contracts only defend via `_transitionState` and `DKG.hasInProgress` gates).

## Audit hot spots

1. **`SYSTEM_CALLER` spoofing.** Any code path where a system tx is constructed without `Recovered::new_unchecked(..., SYSTEM_CALLER)` is an escalation risk. Verify every call site in `onchain_config/` goes through `new_system_call_txn` (`mod.rs:177-195`) and then `transact_system_txn` (`metadata_txn.rs:185-212`).
2. **DKG transcript double-submit / replay.** `finishTransition(transcript)` has no nonce on the contract side; protection relies on consensus + `Reconfiguration._transitionState` + `DKG.hasInProgress`. A reviewer should confirm reth's construction path is gated by reading current state before building the txn (`dkg.rs:374-412`).
3. **Oracle nonce gaps & replays.**
   - *Gaps*: NativeOracle requires strictly increasing, starting at 1; a gap caused by a dropped `ExtraDataType::JWK` is permanent. See also the `ExtraDataType` decoding behaviour described in [gravity-sdk.md](./gravity-sdk.md).
   - *Replays across orphans*: a reconfiguration block that is replaced may cause a cross-chain event to be re-submitted; the on-chain nonce check is the last line of defence. Verify it matches reth-side expectations.
4. **Callback gas exhaustion in NativeOracle.** `record()` succeeds even if the callback reverts. Reviewers should not expect callback side-effects (e.g. JWK store update) to be atomic with `record()`; downstream readers must not assume "record-succeeded ⇒ JWKManager-updated".
5. **Voting-power conversion divergence.** Two code paths (`types.rs` / `dkg.rs`) convert `votingPower` differently; the inconsistency is architecturally non-exploitable but becomes a real bug if contract invariants on stake caps ever break. See [../../design-intent/gravity-reth/validator-set-dkg.md](../../design-intent/gravity-reth/validator-set-dkg.md).
6. **Validator-set read consistency.** `getActiveValidators`, `getPendingActiveValidators`, `getPendingInactiveValidators` and `getNextValidatorSet` must be queried from the same block context when used together (e.g. within `DKG.start`); cross-block reads can produce an inconsistent dealer/target set.
7. **Gas limit for system txns.** Hardcoded `30_000_000` (`mod.rs:188`). DKG transcripts are bounded at 100 MB (`dkg.rs:397`). Any contract logic growth that approaches this limit requires a coordinated gas-limit bump; reviewers adding on-chain logic should estimate the worst-case gas.
8. **Genesis / uninitialised state.** All config contracts assume `requireInitialized()` after genesis. Partial genesis → downstream reads return defaults (0), which can silently corrupt DKG thresholds or epoch intervals. Verify genesis scripts initialise atomically.
