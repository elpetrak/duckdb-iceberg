---
title: "DuckDB-Iceberg Spec Support"
---

# DuckDB-Iceberg Spec Support

This document tracks which features of the [Iceberg Table Spec](https://iceberg.apache.org/spec/) the DuckDB Iceberg extension (`duckdb-iceberg`) supports, per format version. Every entry is checked by reading the spec section, running the existing test(s) and trying the feature in DuckDB directly.

## How to Read This Document

Features are grouped by the format version that introduced them, following [Appendix E: Format version changes](https://iceberg.apache.org/spec/#appendix-e-format-version-changes) and the section layout of the spec.

Each feature has a status for reading and for writing:

| Status | Meaning |
|---|---|
| _supported_ | Works as the spec describes |
| _partial_ | Works with limitations, described in the notes |
| _rejected_ | Not supported, and DuckDB raises a clear error |
| _incorrect_ | Not supported, and DuckDB silently ignores it or returns wrong results |
| _n/a_ | Does not apply (for example, a read-only concept) |
| _unverified_ | Not checked yet |

Test paths are relative to the repository root. `CA/` is short for `test/sql/local/catalog_test_config_setup/catalog_agnostic/`; those tests need a REST catalog (`make mock` is enough for most). Tests under `test/sql/local/` without `CA/` run without any services.

Each entry records the duckdb-iceberg commit it was checked against, because support changes over time.

## Version 1: Analytic Data Tables

### Schemas and Data Types

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _unverified_ | _unverified_ | **Primitive types** | [Primitive Types](https://iceberg.apache.org/spec/#primitive-types) | `test/sql/local/iceberg_scans/iceberg_scan.test`, `test/sql/local/iceberg_fixed_type.test` | |
| _unverified_ | _unverified_ | **Nested types** (`struct`, `list`, `map`) | [Nested Types](https://iceberg.apache.org/spec/#nested-types) | `test/sql/local/nested_types.test`, `CA/irc_nested_types.test` | |
| _unverified_ | _unverified_ | **Field docs** | [Nested Types](https://iceberg.apache.org/spec/#nested-types) | `CA/test_column_doc_comment.test` | |
| _unverified_ | _unverified_ | **Multiple schemas** (`schemas`, `current-schema-id`) | [Table Metadata Fields](https://iceberg.apache.org/spec/#table-metadata-fields) | `test/sql/local/iceberg_scans/iceberg_schemas.test` | |
| _unverified_ | _unverified_ | **Schema evolution**: add, drop, rename columns and fields | [Schema Evolution](https://iceberg.apache.org/spec/#schema-evolution) | `CA/alter/`, `test/sql/local/schema_evolve_struct.test` | |
| _unverified_ | _unverified_ | **Type promotion** (`int`→`long`, `float`→`double`, decimal widening) | [Schema Evolution](https://iceberg.apache.org/spec/#schema-evolution) | `test/sql/local/schema_evolve_int_to_bigint.test`, `test/sql/local/schema_evolve_widen_decimal.test` | |
| _unverified_ | _n/a_ | **Column projection by field id** | [Column Projection](https://iceberg.apache.org/spec/#column-projection) | `test/sql/local/column_mapping.test` | |
| _unverified_ | _unverified_ | **Name mapping** (`schema.name-mapping.default`) | [Name Mapping Serialization](https://iceberg.apache.org/spec/#name-mapping-serialization) | `test/sql/local/native_catalog/name_mapping_reads.test`, `CA/alter/name_mapping_update.test` | |
| _unverified_ | _unverified_ | **Identifier field ids** | [Identifier Field IDs](https://iceberg.apache.org/spec/#identifier-field-ids) | | |

### Partitioning

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _unverified_ | _unverified_ | **Identity transform** | [Partition Transforms](https://iceberg.apache.org/spec/#partition-transforms) | `test/sql/local/partitioning/identity/` | |
| _unverified_ | _unverified_ | **Bucket transform** | [Bucket Transform Details](https://iceberg.apache.org/spec/#bucket-transform-details) | `CA/insert/partitions/bucket/`, `CA/functions/test_iceberg_bucket.test` | |
| _unverified_ | _unverified_ | **Truncate transform** | [Truncate Transform Details](https://iceberg.apache.org/spec/#truncate-transform-details) | `CA/insert/partitions/truncate/`, `CA/functions/test_iceberg_truncate.test` | |
| _unverified_ | _unverified_ | **Temporal transforms** (`year`, `month`, `day`, `hour`) | [Partition Transforms](https://iceberg.apache.org/spec/#partition-transforms) | `CA/delete/partitions/delete_temporal_transforms.test` | |
| _unverified_ | _unverified_ | **Partition evolution** | [Partition Evolution](https://iceberg.apache.org/spec/#partition-evolution) | `CA/alter/test_alter_partitioned_by.test`, `CA/delete/partitions/delete_after_partition_spec_evolution.test` | |
| _unverified_ | _n/a_ | **Partition pruning** | [Scan Planning](https://iceberg.apache.org/spec/#scan-planning) | `CA/insert/partitions/multi_insert/test_manifest_skip_combined_transforms.test` | |

### Sorting

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _n/a_ | _unverified_ | **Sort orders** | [Sorting](https://iceberg.apache.org/spec/#sorting) | `CA/alter/sorted_by/` | |

### Snapshots

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _unverified_ | _n/a_ | **Time travel by snapshot id and timestamp** | [Point in Time Reads](https://iceberg.apache.org/spec/#point-in-time-reads-time-travel) | `test/sql/local/time_travel_after_last_snapshot.test`, `test/sql/local/time_travel_before_first_snapshot.test` | |
| _unverified_ | _unverified_ | **Branches and tags** (`refs`) | [Snapshot References](https://iceberg.apache.org/spec/#snapshot-references) | `test/sql/local/iceberg_scans/iceberg_metadata.test` (data: `data/persistent/snapshot_refs`), `test/sql/local/native_catalog/snapshot_ref_retention.test` | |
| _n/a_ | _unverified_ | **Rollback to snapshot** | [Snapshots](https://iceberg.apache.org/spec/#snapshots) | `CA/maintenance/rollback_to_snapshot.test` | |
| _n/a_ | _unverified_ | **Snapshot retention / expiration** | [Snapshot Retention Policy](https://iceberg.apache.org/spec/#snapshot-retention-policy) | | |

### Table Metadata and Manifests

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _unverified_ | _unverified_ | **v1 tables** | [Version 1](https://iceberg.apache.org/spec/#version-1-analytic-data-tables) | `test/sql/local/test_v1_manifest_entry.test`, `CA/test_v1_unsupported_writes.test` | |
| _unverified_ | _n/a_ | **File system tables** (`version-hint.text`) | [File System Tables](https://iceberg.apache.org/spec/#file-system-tables) | `test/sql/local/iceberg_scans/iceberg_scan.test` | |
| _unverified_ | _n/a_ | **GZIP-compressed metadata** | [Naming for GZIP compressed Metadata JSON files](https://iceberg.apache.org/spec/#naming-for-gzip-compressed-metadata-json-files) | | |
| _unverified_ | _unverified_ | **Column metrics and bounds** | [Field-level Metrics and Statistics](https://iceberg.apache.org/spec/#field-level-metrics-and-statistics) | `test/sql/local/null_stats.test`, `test/sql/local/generated_bounds.test` | |
| _unverified_ | _unverified_ | **Table statistics** (Puffin) | [Table Statistics](https://iceberg.apache.org/spec/#table-statistics) | | |

## Version 2: Row-level Deletes

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _unverified_ | _unverified_ | **Sequence numbers and inheritance** | [Sequence Number Inheritance](https://iceberg.apache.org/spec/#sequence-number-inheritance) | `CA/maintenance/rewrite_manifest_inheritance_upgrade.test` | |
| _unverified_ | _unverified_ | **Position delete files** | [Position Delete Files](https://iceberg.apache.org/spec/#position-delete-files) | `CA/delete/test_positional_delete_filename_bounds.test` | check whether DuckDB writes them for v2 tables |
| _unverified_ | _unverified_ | **Equality delete files** | [Equality Delete Files](https://iceberg.apache.org/spec/#equality-delete-files) | `test/sql/local/equality_deletes.test`, `CA/delete/equality_deletes/` | |
| _n/a_ | _unverified_ | **`DELETE`** | [Row-level Deletes](https://iceberg.apache.org/spec/#row-level-deletes) | `CA/delete/` | |
| _n/a_ | _unverified_ | **`UPDATE`** | [Row-level Deletes](https://iceberg.apache.org/spec/#row-level-deletes) | `CA/update/` | |
| _n/a_ | _unverified_ | **`MERGE INTO`** | [Row-level Deletes](https://iceberg.apache.org/spec/#row-level-deletes) | `CA/merge/` | |
| _n/a_ | _unverified_ | **Optimistic concurrency / commit retry** | [Commit Conflict Resolution and Retry](https://iceberg.apache.org/spec/#commit-conflict-resolution-and-retry) | `CA/transactions/` | |

## Version 3: Extended Types and Capabilities

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _unverified_ | _unverified_ | **Default values** (`initial-default`, `write-default`) | [Default values](https://iceberg.apache.org/spec/#default-values) | `test/sql/local/initial_default_all_types.test`, `CA/create/default/`, `CA/alter/alter_default.test` | |
| _unverified_ | _unverified_ | **Nanosecond timestamps** (`timestamp_ns`, `timestamptz_ns`) | [Primitive Types](https://iceberg.apache.org/spec/#primitive-types) | `CA/alter/alter_type_timestamptz_ns_v3.test`, `CA/insert/partitions/timestamptz_ns/` | |
| _unverified_ | _unverified_ | **`unknown` type** | [Primitive Types](https://iceberg.apache.org/spec/#primitive-types) | `CA/create/test_create_unknown_type.test`, `CA/insert/test_unknown_type_omitted_from_data_files.test` | |
| _unverified_ | _unverified_ | **`variant` type** | [Semi-structured Types](https://iceberg.apache.org/spec/#semi-structured-types) | `CA/insert/iceberg_types/variant/`, `CA/file_properties/test_parquet_shred_variants.test` | |
| _unverified_ | _unverified_ | **`geometry` / `geography` types** | [Primitive Types](https://iceberg.apache.org/spec/#primitive-types), [Appendix G](https://iceberg.apache.org/spec/#appendix-g-geospatial-notes) | `CA/insert/iceberg_types/test_geometry_type.test`, `test/sql/local/native_catalog/geography_column.test` | |
| _unverified_ | _unverified_ | **Deletion vectors** | [Deletion Vectors](https://iceberg.apache.org/spec/#deletion-vectors) | `test/sql/local/test_basic_deletion_vectors.test`, `CA/delete/test_grouped_deletion_vectors.test`, `CA/delete/test_delete_dv_supersedes_position_delete_file.test` | |
| _unverified_ | _unverified_ | **Row lineage** (`_row_id`, `_last_updated_sequence_number`, `next-row-id`) | [Row Lineage](https://iceberg.apache.org/spec/#row-lineage) | `test/sql/local/row_lineage_stored_columns.test`, `CA/test_row_lineage_write.test`, `CA/test_row_lineage_next_row_id.test` | |
| _unverified_ | _unverified_ | **Multi-argument transforms** (`source-ids`) | [Partition Specs](https://iceberg.apache.org/spec/#partition-specs) | `CA/alter/test_partition_field_id_reuse_schema.test` | |
| _unverified_ | _unverified_ | **Encryption keys** (`encryption-keys`, `key-id`) | [Encryption Keys](https://iceberg.apache.org/spec/#encryption-keys) | `test/sql/local/iceberg_scans/encrypted_table.test`, `CA/table_properties/test_reject_encryption_properties.test` | |
| _n/a_ | _unverified_ | **Upgrading v2 → v3** | [Appendix E: Version 3](https://iceberg.apache.org/spec/#version-3) | `CA/alter/table_properties/test_upgrade_format_version_2_to_3.test` | |

## Version 4: Metadata Structure and Representation

**Version 4 is under active development and has not been formally adopted.**

| Read | Write | Feature | Spec | Test | Notes |
|---|---|---|---|---|---|
| _unverified_ | _unverified_ | **v4 tables** | [Version 4](https://iceberg.apache.org/spec/#version-4-metadata-structure-and-representation) | `test/sql/local/iceberg_scans/unsupported_format_version.test` (data: `data/persistent/format_version_4`) | |
| _unverified_ | _unverified_ | **Relative paths and optional `location`** | [Paths in Metadata](https://iceberg.apache.org/spec/#paths-in-metadata) | | |
| _n/a_ | _unverified_ | **No new equality deletes** | [Appendix E: Version 4](https://iceberg.apache.org/spec/#version-4) | | |
| _unverified_ | _unverified_ | **Constraints / `primary-key`** | [Constraints](https://iceberg.apache.org/spec/#constraints) | `CA/create/test_create_table_unsupported_constraints.test` | |
| _n/a_ | _unverified_ | **Strictly increasing snapshot `timestamp-ms`** | [Appendix E: Version 4](https://iceberg.apache.org/spec/#version-4) | | |
| _unverified_ | _unverified_ | **`tracked_file` manifests and manifest deletion vectors** | [Manifest Schema](https://iceberg.apache.org/spec/#manifest-schema), [Manifest Deletion Vectors](https://iceberg.apache.org/spec/#manifest-deletion-vectors) | | |
| _unverified_ | _unverified_ | **Content stats** | [Content Stats](https://iceberg.apache.org/spec/#content-stats) | | |

## Session Log

One entry per session, newest first. Copy this template:

```
### YYYY-MM-DD: <feature>

- Spec: <section link>
- duckdb-iceberg commit: <sha>
- Existing test(s): <path> (pass/fail)
- What I tried: <SQL or steps>
- Result: <status> because <reason>
- Follow-up: <issue link / PR idea / none>
```
