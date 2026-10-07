# Configuration Contracts

This folder contains the input interface contract documentation for S-CORE modules as they integrate with the Vehicle Evidence Layer (VEL). The configuration contracts consist of three sub-contracts: Input Interface Contract, Normalization Contract, and Output Evidence Contract.

## Contents

```{toctree}
:maxdepth: 2

input-interface-contract/s-core-module-input-design
input-interface-contract/s-core-modules/lifecycle-input-design
input-interface-contract/input-schema-structure
input-interface-contract/source-identity-and-timestamp
input-interface-contract/validation-failure-handling
normalization-contract/mapping-rule-model
normalization-contract/unmapped-and-unknown-value-handling
normalization-contract/mapping-quality-outputs
output-evidence-contract/normalized-evidence-record-structure
output-evidence-contract/evidence-quality-metadata
output-evidence-contract/traceability-and-correlation-metadata
```

## Related Package

The canonical interface artifacts for the S-CORE Lifecycle module live in the package directory: `interfaces/score-evidence-layer-package/` (schemas, mappings, collector config, tests, and signatures). The vendor-NPU reference package is in `interfaces/vendor-a-npu-evidence-layer-package/`, and the output evidence package is in `interfaces/vel-output-evidence-package/`.