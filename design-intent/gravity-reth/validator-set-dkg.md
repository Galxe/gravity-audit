# Validator Set & DKG — Voting Power Conversion

> Why the two voting-power conversion paths (ValidatorSet read vs. DKG read) differ in their overflow handling, and why the theoretical divergence is not exploitable.

## Scope
- `crates/pipe-exec-layer-ext-v2/execute/src/onchain_config/types.rs` (~line 114) — `convert_validator_consensus_info` / `wei_to_ether`.
- `crates/pipe-exec-layer-ext-v2/execute/src/onchain_config/dkg.rs` (~lines 265-268) — `convert_validator` voting-power conversion.
- Keywords: `votingPower`, `wei_to_ether`, `u64`, `U256`, `DKG`, `ValidatorSet`, `ValidatorConsensusInfo`, `convert_validator`.

## Design Intent

- **Voting power is denominated in whole ether on the consensus layer and fits in `u64`.** The system's tokenomics and staking bounds guarantee no single validator's stake can exceed `u64::MAX` ether (`≈ 1.8 × 10^19` ETH). The staking contracts enforce caps far below this threshold.
- Any `votingPower` above `u64::MAX` ether is, by definition, a symptom of a contract bug or a malformed governance migration — not a value that a well-formed validator set should ever contain.

## Invariants
- For every `ValidatorConsensusInfo` read from the validator-management contract, `votingPower / 10^18 ≤ u64::MAX`.
- Both the ValidatorSet read path and the DKG read path receive the same `ValidatorConsensusInfo` values from the same contract state within a single block, so their conversions operate on identical inputs.

## Known Non-Issues

### Voting power conversion inconsistent between ValidatorSet and DKG paths — closed-as-intent, ref [#8](https://github.com/Galxe/gravity-audit/issues/8)
- **Surface finding.** Two code paths convert `U256` wei → consensus-layer `u64` ether differently:
  - `types.rs:107-114` uses `wei_to_ether(info.votingPower).to::<u64>()`, which per alloy's documented semantics **panics** on overflow.
  - `dkg.rs:264-268` uses `(voting_power / 10^18).try_into().unwrap_or(u64::MAX)`, which **saturates** to `u64::MAX`.

  For the same `votingPower > u64::MAX * 10^18` wei, one path crashes the node while the other returns a different value — a latent inconsistency and a single-validator-can-DoS vector on the ValidatorSet read path.
- **Why this is not a bug.** Voting power is architecturally bounded and cannot exceed `u64::MAX` ether. The staking contracts enforce validator stake limits far below that threshold. For any well-formed validator set coming off the chain, both conversion paths produce the same value; the divergence only appears for inputs that, by contract invariant, cannot exist in production. The cosmetic inconsistency between `.to::<u64>()` and `try_into().unwrap_or(u64::MAX)` is acknowledged code smell but not a live bug.
- **Boundary — when this would become a real bug.**
  - If any governance migration, emergency contract patch, or bug ever produces a `ValidatorConsensusInfo` with `votingPower > u64::MAX * 10^18` wei, the ValidatorSet path will crash the node — which then *is* a real production issue even though the surface inconsistency is not.
  - If the consensus-layer voting-power type is ever widened (e.g. to `u128`), both paths must be updated together; having them drift further is a correctness risk.
  - If a new third code path is added to convert voting power, it must use the same helper as the existing two to avoid compounding the smell.
  - Recommended cleanup when touched: introduce a single `wei_to_validator_power_u64` helper shared by both call sites, saturating with a `warn!` log — this eliminates the divergence without changing observable behaviour.
- **Reference.** Issue [#8](https://github.com/Galxe/gravity-audit/issues/8), closing comment: _"reject: voting power is impossible large than u64::MAX"._
