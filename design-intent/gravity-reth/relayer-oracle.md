# Relayer / Oracle Manager — Delivery & Persistence Semantics

> Why the relayer's cursor/last-processed state uses `AtomicU64` + `Mutex` without stronger ordering, and why swallowing persistence failures while advancing in-memory state is safe.

## Scope
- `crates/pipe-exec-layer-ext-v2/relayer/src/oracle_manager.rs` — `OracleRelayerManager`, `poll`, `poll_uri`, `fast_forward`, `set_cursor`, `set_last_processed`, `update_and_save_state`.
- `crates/pipe-exec-layer-ext-v2/relayer/` — `BlockchainEventSource`, persistence via `relayer_state.json`.
- Keywords: `relayer`, `oracle_manager`, `cursor`, `last_processed`, `Relaxed`, `AtomicU64`, `Mutex`, `fast_forward`, `poll`, `poll_uri`, `relayer_state.json`, `persistence`, `double-spend`, `replay`.

## Design Intent

- **The relayer is a read-only data feeder, not the source of truth.** It reads cross-chain events (e.g. from an Ethereum RPC via `eth_getLogs`) and forwards them to the consensus layer. The consensus layer (gravity-sdk) holds the authoritative cursor/nonce on-chain and is the deduplication boundary.
- **Serial per-URI calling model.** `poll_uri` for any given URI is always called serially by the manager — never concurrently with itself. Within one `poll_uri` call the execution is strictly sequential (`fast_forward(nonce, block)` → `source.poll()`). There is no external observer of the intermediate state between `set_last_processed` and `set_cursor`.
- **Persisted `relayer_state.json` is a restart optimisation, not a correctness input.** It lets the relayer avoid re-scanning blocks on restart; losing it (or having it lag) only means slightly more work (and some harmless re-delivery) on next boot.
- **At-least-once delivery + downstream deduplication.** The pattern is deliberately at-least-once: if the relayer re-delivers an event that the consensus layer has already processed, consensus drops it by nonce.

## Invariants
- `poll_uri` for a given URI is never invoked concurrently.
- The consensus layer dedupes cross-chain messages by nonce against its on-chain state.
- Re-delivery of an already-processed event is harmless at the consensus layer.
- `relayer_state.json` freshness is best-effort: a stale-on-disk checkpoint means extra re-scans on restart, nothing worse.

## Known Non-Issues

### Relayer cursor and last_processed state updated non-atomically with Relaxed ordering — closed-as-intent, ref [#6](https://github.com/Galxe/gravity-audit/issues/6)
- **Surface finding.** `cursor: AtomicU64` and `last_processed: Mutex<…>` are separate fields updated with `Relaxed` ordering. A reader observing an intermediate state (`last_processed` advanced, `cursor` not yet) would see an inconsistent view; a crash between the two updates would leave the relayer "forgetting" what it just delivered; `Relaxed` provides no cross-thread ordering guarantees for this pair.
- **Why this is not a bug.**
  - *Crash window*: both fields are in-memory only. If the process crashes between them, both are lost; recovery comes from `relayer_state.json` plus the on-chain nonce held by the consensus layer, which will drop any duplicates on the next poll. Correctness is unaffected.
  - *Race window*: the described race requires **concurrent** `poll_uri` calls for the same URI, which by the serial-per-URI model never happens. The inconsistency window is unobservable.
  - *`Relaxed` ordering*: `Relaxed` only causes problems under concurrent multi-threaded access. A single task always sees its own prior writes; Tokio's task-resume path inserts the memory barriers needed when a task resumes on a different core after an `.await`.
  - The separated `AtomicU64` + `Mutex` shape is not the prettiest (could be unified into one struct), but it is functionally correct under the actual calling model.
- **Boundary — when this would become a real bug.**
  - If the design ever allows **concurrent** `poll_uri` calls for the same URI (e.g. parallel polling of the same source, speculative prefetch, retry overlapping with the primary path), the Relaxed/non-atomic pair stops being safe and must be replaced with a single `Mutex`-guarded struct or `AcqRel`-ordered atomics.
  - If the consensus layer is ever removed as the authoritative dedup boundary (e.g. a downstream consumer that cannot tolerate duplicate delivery is added), at-least-once stops being sufficient.
- **Reference.** Issue [#6](https://github.com/Galxe/gravity-audit/issues/6).

### Relayer persistence failure silently swallowed allowing unbounded replay window — closed-as-intent, ref [#7](https://github.com/Galxe/gravity-audit/issues/7)
- **Surface finding.** In `update_and_save_state`, when `candidate.save(&path)` fails, the code logs a warning and still assigns `*state = candidate` — in-memory state advances even though the on-disk checkpoint did not. If the process restarts after a sustained disk failure, `relayer_state.json` can lag arbitrarily behind reality, allowing re-delivery of events that were already forwarded — framed in the report as a double-spend risk on bridged assets.
- **Why this is not a bug.**
  - The relayer is a **read-only feeder**; the authoritative cursor lives on-chain in the consensus layer. `relayer_state.json` is purely a fast-restart optimisation.
  - On restart with a stale checkpoint, the relayer re-reads events it already delivered. Consensus sees those events have nonces `≤` the on-chain nonce and drops them as already-processed. No double-delivery reaches the application layer.
  - There is **no double-spend**: the consensus layer, not the relayer, decides whether to process a cross-chain message. Duplicate delivery from the relayer is harmless.
  - Advancing in-memory state on save failure is the intentional trade-off (see the comment around `oracle_manager.rs:307-311`): it avoids *duplicate delivery within the running process* while the disk is unavailable. The worst case (restart after sustained disk failure) is harmless re-delivery that consensus dedupes.
- **Boundary — when this would become a real bug.**
  - If a downstream consumer of the relayer's output is ever added that **cannot** tolerate at-least-once delivery (i.e. does not dedupe by nonce on-chain), the save-failure-swallow becomes unsafe.
  - If the relayer is repurposed as an authoritative source (writes directly to user-visible state without a consensus-layer dedupe in front), the persistence failure must become a hard error.
  - If the on-chain nonce is ever not monotonic per source, the consensus-layer dedup assumption breaks.
- **Reference.** Issue [#7](https://github.com/Galxe/gravity-audit/issues/7).
