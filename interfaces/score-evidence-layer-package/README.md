# S-CORE Lifecycle RawEvidenceSchema — Evidence Interface Package

This package contains the configuration artifacts for the S-CORE **Lifecycle** module that serves as a VEL data source. These conform to the Task #38 Input Schema Structure, Task #39 Source Identity & Timestamp, and Task #42-#44 Normalization Mapping contracts.

## Structure

```text
score-evidence-layer-package/
├── manifest.yaml
├── README.md
├── collectors/
│   └── lifecycle-collector.yaml
├── schemas/
│   └── score-lifecycle-state-v1.0.0.yaml
├── mappings/
│   └── score-lifecycle-to-evidence-v1.0.0.yaml
├── tests/
│   └── lifecycle-normalization-vectors.yaml
└── signatures/
    ├── content-digests.yaml
    └── manifest.sig
```

## Files

| File | Kind | Description |
| --- | --- | --- |
| `collectors/lifecycle-collector.yaml` | `VehicleEvidenceCollector` | Lifecycle (Launch Manager / Health Monitor) collector config. |
| `schemas/score-lifecycle-state-v1.0.0.yaml` | `RawEvidenceSchema` | Lifecycle (Launch Manager / Health Monitor) input schema. |
| `mappings/score-lifecycle-to-evidence-v1.0.0.yaml` | `EvidenceNormalizationRule` | Lifecycle → normalized evidence mapping. |
| `tests/lifecycle-normalization-vectors.yaml` | `NormalizationTestSuite` | Lifecycle normalization verification vectors. |
| `signatures/content-digests.yaml` | `IntegrityManifest` | SHA-256 digests of all package artifacts. |
| `signatures/manifest.sig` | `Signature` | Manifest signature placeholder (must be signed in production). |

## Are the Normalized / Output Evidence YAMLs shared?

**Yes.** The `NormalizedEvidenceSchema` and the Output Evidence schema are **shared across all sources** — they are NOT module-specific.

- The single shared output schema is [`score-normalized-evidence-v1.0.0.yaml`](../vendor-a-npu-evidence-layer-package/schemas/score-normalized-evidence-v1.0.0.yaml). Every source (vendor-NPU, S-CORE lifecycle, etc.) normalizes into this same schema.
- Each source only needs its own **`RawEvidenceSchema`** (input) and its own **`EvidenceNormalizationRule`** (mapping). You do NOT create separate normalized/evidence/output YAMLs per module.

So for the S-CORE Lifecycle module, the complete set is exactly three files:
1. `VehicleEvidenceCollector` — module-specific collector config (in `collectors/`).
2. `RawEvidenceSchema` — module-specific input contract (in `schemas/`).
3. `EvidenceNormalizationRule` — module-specific mapping to the shared normalized evidence (in `mappings/`).

## Verification Vectors

The S-CORE Lifecycle module has its own normalization verification vectors under `tests/`, capturing the expected input/output pairs for the module's mapping rule:

| Test File | Mapping Under Test |
| --- | --- |
| `tests/lifecycle-normalization-vectors.yaml` | `score-lifecycle-to-evidence` |

## Signatures

The `signatures/content-digests.yaml` contains the real SHA-256 digests of all package artifacts for integrity verification. The `signatures/manifest.sig` is a placeholder indicating the package is unsigned; production packages must be signed by the approved S-CORE release process.