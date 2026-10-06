# VEL Output Evidence Package

This package contains the **single shared** `NormalizedEvidenceSchema` (`score-normalized-evidence-v1.0.0.yaml`) that every VEL source package normalizes into. It conforms to the Output Evidence Contract (Feature #45: `normalized-evidence-record-structure.md`, `evidence-quality-metadata.md`, `traceability-and-correlation-metadata.md`).

## Why this is a separate package

The normalized Vehicle Evidence output contract is **not owned by any single source**. It is the common target schema that all source-specific `EvidenceInterfacePackage`s (e.g. [`score-evidence-layer-package`](../score-evidence-layer-package/), [`vendor-a-npu-evidence-layer-package`](../vendor-a-npu-evidence-layer-package/)) reference via `outputSchemaRef` (in collector configs) and `sharedOutputSchema` (in package manifests). Keeping it in its own package avoids implying ownership by any one vendor or module, and avoids duplicating the same schema file across every source package.

## Structure

```text
vel-output-evidence-package/
├── manifest.yaml
├── README.md
├── schemas/
│   └── score-normalized-evidence-v1.0.0.yaml
└── signatures/
    ├── content-digests.yaml
    └── manifest.sig
```

## Files

| File | Kind | Description |
| --- | --- | --- |
| `schemas/score-normalized-evidence-v1.0.0.yaml` | `NormalizedEvidenceSchema` | The normalized Vehicle Evidence record contract: header, identity, observation, time, quality, and traceability sections. |
| `signatures/content-digests.yaml` | `IntegrityManifest` | SHA-256 digests of all package artifacts. |
| `signatures/manifest.sig` | `Signature` | Manifest signature placeholder (must be signed in production). |

## How source packages reference this schema

Each source-specific collector config sets `outputSchemaRef.path` to this package's schema file (relative path depends on the collector's location), for example:

```yaml
outputSchemaRef:
  name: score-normalized-evidence
  version: 1.0.0
  path: ../vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml
```

Each source package's top-level `manifest.yaml` also declares this as its `sharedOutputSchema`:

```yaml
sharedOutputSchema:
  path: ../vel-output-evidence-package/schemas/score-normalized-evidence-v1.0.0.yaml
  role: output-schema
```

## Responsibility Boundary

- **Evidence Layer, S-CORE**: owns this shared output schema; defines quality, traceability, and record structure rules.
- **Source packages** (vendor runtime, S-CORE modules): own their own `RawEvidenceSchema` (input) and `EvidenceNormalizationRule` (mapping); they normalize into this shared schema but do not define it.
- **Orchestrator Layer, S-CORE** / **OEM Policy Tier**: consume normalized evidence; they do not influence this schema's structure.
