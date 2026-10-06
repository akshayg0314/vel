# Design Documentation

This folder contains design documentation for the Vehicle Evidence Layer (VEL), including both generic VEL design documents (input interface contracts, collector design patterns) and module-specific integration designs.

The folder structure mirrors the VEL draft documentation structure. Currently, only the S-CORE Lifecycle module documentation is included.

## Contents

```{toctree}
:maxdepth: 2

configuration-contracts/README
evidence-ingestion/README
data-field-analysis/README
```

## Common References

- [VEL glossary](../glossary_eng.rst): Vehicle Evidence, Source, Source Collector, Input Interface Definition, Normalization Mapping, Evidence Quality, S-CORE Module, Evidence Runtime, Designated Consumer, Correlation Identifier, VEL Health, Output Evidence Definition.
- [VEL architectural design draft](../vel_architectural_design_draft_eng.md): VEL Evidence Layer principles and boundaries.
- [Requirements](../requirements_eng.rst): FR-VEL-003, FR-VEL-008, FR-VEL-012, FR-VEL-013, FR-VEL-014, FR-VEL-015, AOU-VEL-002, saf_req_vel_*.

## Scope and Boundaries

- VEL is an Evidence producer. It does not perform multi-node coordination, lifecycle execution, process/container/hardware control, or final OEM decisions.
- Platform-specific collectors are separate from the common Evidence Layer.
