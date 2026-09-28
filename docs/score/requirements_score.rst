.. _vel_score_requirements:

VEL Requirements (S-CORE Style)
===============================

This Sphinx-Needs view mirrors the current VEL requirements. Functional, security, and safety requirements are intentionally separated.

Stakeholder Requirements
------------------------

.. stkh_req:: Normalized Vehicle Evidence Format
   :id: stkh_req__vel__normalized_format

   The platform shall define a normalized Vehicle Evidence data format.

.. stkh_req:: Metric Collection
   :id: stkh_req__vel__metric_collection

   The platform shall collect configured runtime and hardware metrics.

.. stkh_req:: Accessible Metric Source Mechanism
   :id: stkh_req__vel__accessible_metric_source

   The platform shall use accessible source mechanisms for metric collection.

.. stkh_req:: S-CORE Module State Collection
   :id: stkh_req__vel__score_state_collection

   The platform shall collect configured S-CORE module state information.

.. stkh_req:: Source Normalization
   :id: stkh_req__vel__source_normalization

   The platform shall normalize source-specific data representations.

.. stkh_req:: Source Identity Traceability
   :id: stkh_req__vel__source_identity

   The platform shall preserve source identity for collected evidence.

.. stkh_req:: Observation Time Traceability
   :id: stkh_req__vel__observation_time

   The platform shall preserve observation time for collected evidence.

.. stkh_req:: Correlation Traceability
   :id: stkh_req__vel__correlation

   The platform shall preserve correlation information when source context provides it.

.. stkh_req:: Evidence Quality
   :id: stkh_req__vel__evidence_quality

   The platform shall expose evidence quality information.

.. stkh_req:: VEL Health
   :id: stkh_req__vel__health

   The platform shall expose VEL health information.

.. stkh_req:: External Evidence Access
   :id: stkh_req__vel__external_access

   The platform shall expose normalized evidence to designated consumers.

.. stkh_req:: Platform Collector Separation
   :id: stkh_req__vel__collector_separation

   The platform shall separate platform-specific collectors from the common Evidence Layer.

.. stkh_req:: Input Interface Configuration
   :id: stkh_req__vel__input_interface_configuration

   The platform shall define source input interfaces through configuration.

.. stkh_req:: Output Evidence Interface Configuration
   :id: stkh_req__vel__output_interface_configuration

   The platform shall define normalized evidence output interfaces through configuration.

.. stkh_req:: Normalization Mapping Configuration
   :id: stkh_req__vel__normalization_mapping_configuration

   The platform shall define source-to-evidence normalization mappings through configuration.

.. stkh_req:: Aggregation Data Format
   :id: stkh_req__vel__aggregation_format

   The platform shall support a consistent aggregation data format.

Feature Requirements
--------------------

.. feat_req:: Metric Collection
   :id: feat_req__vel__metric_collection
   :satisfies: stkh_req__vel__metric_collection

   The system shall collect configured runtime and hardware metrics.

.. feat_req:: Accessible Metric Source Mechanism
   :id: feat_req__vel__accessible_metric_source
   :satisfies: stkh_req__vel__accessible_metric_source

   The system shall collect metric data through files, commands, or supported operating-system interfaces.

.. feat_req:: S-CORE Module State Collection
   :id: feat_req__vel__score_state_collection
   :satisfies: stkh_req__vel__score_state_collection

   The system shall collect configured S-CORE module state information through S-CORE APIs exposed by the deployed environment.

.. feat_req:: Normalized Vehicle Evidence Format
   :id: feat_req__vel__normalized_format
   :satisfies: stkh_req__vel__normalized_format

   The system shall transform collected source data into a normalized Vehicle Evidence data format.

.. feat_req:: Source Normalization
   :id: feat_req__vel__source_normalization
   :satisfies: stkh_req__vel__source_normalization

   The system shall normalize source-specific units and representations according to the configured normalization mapping.

.. feat_req:: Source Identity Metadata
   :id: feat_req__vel__source_identity
   :satisfies: stkh_req__vel__source_identity

   The system shall include source identity in each evidence record.

.. feat_req:: Observation Time Metadata
   :id: feat_req__vel__observation_time
   :satisfies: stkh_req__vel__observation_time

   The system shall include observation timestamp in each evidence record.

.. feat_req:: Correlation Metadata
   :id: feat_req__vel__correlation
   :satisfies: stkh_req__vel__correlation

   The system shall include a correlation identifier in each evidence record when source context provides it.

.. feat_req:: Evidence Quality Metadata
   :id: feat_req__vel__quality
   :satisfies: stkh_req__vel__evidence_quality

   The system shall attach quality information to each evidence record or evidence batch.

.. feat_req:: VEL Health Exposure
   :id: feat_req__vel__health
   :satisfies: stkh_req__vel__health

   The system shall expose VEL health information.

.. feat_req:: External Evidence Interface
   :id: feat_req__vel__external_interface
   :satisfies: stkh_req__vel__external_access

   The system shall expose normalized evidence through an external interface for designated consumers.

.. feat_req:: Platform Collector Separation
   :id: feat_req__vel__collector_separation
   :satisfies: stkh_req__vel__collector_separation

   The system shall isolate OS-specific and platform-specific collectors from the common Evidence Layer.

.. feat_req:: Input Interface Configuration
   :id: feat_req__vel__input_interface_configuration
   :satisfies: stkh_req__vel__input_interface_configuration

   The system shall load source input interface definitions from configuration.

.. feat_req:: Output Evidence Interface Configuration
   :id: feat_req__vel__output_interface_configuration
   :satisfies: stkh_req__vel__output_interface_configuration

   The system shall load normalized evidence output interface definitions from configuration.

.. feat_req:: Normalization Mapping Configuration
   :id: feat_req__vel__normalization_mapping_configuration
   :satisfies: stkh_req__vel__normalization_mapping_configuration

   The system shall load source-to-evidence normalization mapping rules from configuration.

.. feat_req:: Aggregation Data Format
   :id: feat_req__vel__aggregation_data_format
   :satisfies: stkh_req__vel__aggregation_format

   The system shall support an aggregation data format agreed with responsible data-format stakeholders.

Security Requirements
---------------------

.. sec_req:: Security Mechanism Integration Boundary
   :id: sec_req__vel__interface_policy

   VEL shall apply security mechanisms selected by the deployed environment to the external evidence interface.

.. sec_req:: Authentication Authority Boundary
   :id: sec_req__vel__authentication_boundary

   VEL shall not define or operate an independent authentication authority.

.. sec_req:: Evidence Interface Authentication
   :id: sec_req__vel__interface_authentication

   When caller authentication is required by the deployed environment, VEL shall rely on caller identity information supplied by the deployed environment.

.. sec_req:: Evidence Interface Access Control
   :id: sec_req__vel__interface_access_control

   VEL shall restrict external evidence access according to consumer access rules configured by the deployed environment.

.. sec_req:: Evidence Interface Integrity
   :id: sec_req__vel__interface_integrity

   VEL shall support the integrity protection method selected by the deployed environment for evidence exchanged through the external evidence interface.

Safety Requirements
-------------------

.. saf_req:: Lifecycle Execution Boundary
   :id: saf_req__vel__lifecycle_boundary

   VEL shall not execute workload lifecycle operations.

.. saf_req:: Process Control Boundary
   :id: saf_req__vel__process_control_boundary

   VEL shall not execute process control operations.

.. saf_req:: Container Control Boundary
   :id: saf_req__vel__container_control_boundary

   VEL shall not execute container control operations.

.. saf_req:: Hardware Control Boundary
   :id: saf_req__vel__hardware_control_boundary

   VEL shall not execute hardware control operations.

.. saf_req:: Multi-Node Coordination Boundary
   :id: saf_req__vel__multi_node_boundary

   VEL shall not execute multi-node coordination.

.. saf_req:: OEM Decision Boundary
   :id: saf_req__vel__oem_decision_boundary

   VEL shall not execute final OEM vehicle decision logic.