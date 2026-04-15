# gravity-reth — Dependencies

Dependency-interaction notes for [gravity-reth](https://github.com/Galxe/gravity-reth).

| File | Dependency | Role |
|------|------------|------|
| [gravity-sdk.md](./gravity-sdk.md) | [Galxe/gravity-sdk](https://github.com/Galxe/gravity-sdk) | Consensus layer (BFT, aptos-core based). Drives block ordering & finality; gravity-reth executes the blocks it orders. |
| [gravity_chain_core_contracts.md](./gravity_chain_core_contracts.md) | Galxe/gravity_chain_core_contracts | On-chain Solidity system contracts (ValidatorManagement, Reconfiguration, DKG, NativeOracle, Blocker, Timestamp, …). gravity-reth calls these via system transactions and reads them via static calls. |

See [../README.md](../README.md) for the file format contract.
