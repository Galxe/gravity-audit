# gravity-reth — Design Intent

Design-intent notes for [gravity-reth](https://github.com/Galxe/gravity-reth), the execution layer for Gravity Chain (a fork of `reth`).

## Modules

| File | Scope |
|------|-------|
| [consensus-epoch.md](./consensus-epoch.md) | Epoch transitions, DKG-triggered reconfiguration, block body construction in the reconfiguration path. |
| [block-execution.md](./block-execution.md) | Block-execution pipeline error handling, panic semantics, system-transaction failure modes. |
| [relayer-oracle.md](./relayer-oracle.md) | Cross-chain relayer / oracle manager — cursor & nonce state, persistence, delivery semantics. |
| [validator-set-dkg.md](./validator-set-dkg.md) | Validator set loading, voting-power conversion, DKG transcript construction. |

See [../README.md](../README.md) for the file format contract.
