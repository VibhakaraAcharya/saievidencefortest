# Trace360 Phase 1 Plan

## Summary
Build Trace360 as a Python, CLI-first evidence collection engine that reads existing audit-control YAMLs from `/controls/`, executes scheduled-run-compatible control sets, collects GCP evidence, stores raw payloads in GCS, and writes searchable metadata to PostgreSQL. Phase 1 is the backbone only: no UI, no MCP, and no cloud scheduler wiring yet, but the runner and packaging should be shaped so Cloud Run Job execution is the next straightforward step.

## Key Changes
- Scaffold a Python package with these main areas: `core` for models/loader/runner/evaluator/registry/config, `connectors/gcp` for collectors and normalizers, `storage` for GCS/Postgres adapters, and `interfaces/cli` for operator commands.
- Move current YAML controls from `yamls/` into `/controls/` without breaking their schema. Preserve fields like `id`, `request_id`, `connector`, `operation`, `scope`, `expected`, `evaluation`, and `evidence`.
- Add a control-set YAML format for grouped runs and implement both `run-control` and `run-control-set` CLI commands. Single-control execution remains useful for debugging; control-set execution is the primary scheduled-run contract.
- Implement an internal canonical model for `ControlDefinition`, `ControlSet`, `EvidenceRecord`, and `RunRecord`. The loader should accept the current YAML shape and normalize it into typed internal objects rather than rewriting source files.
- Build a connector registry that resolves `connector + operation` to callable collectors. The GCP connector layer should be generic enough to support all current GCP YAML controls, while the first fully wired path uses Cloud Storage bucket-related controls.
- Keep connector responsibilities narrow: authenticate, fetch raw source data, and return structured payloads. Put normalization, evaluation, hashing, run bookkeeping, and persistence in the engine layer.
- Persist raw evidence JSON to GCS with deterministic object naming that includes run timestamp, connector, and control ID. Persist metadata rows to PostgreSQL with fields at minimum for `run_id`, `collected_at`, `control_id`, `request_id`, `connector`, `operation`, `resource_id`, `status`, `severity`, `category`, `raw_evidence_uri`, `normalized_result`, and `evidence_hash`.
- Implement evaluation logic that compares normalized output against `expected` and applies optional `evaluation` rules from the YAML. The result should emit `PASS`, `FAIL`, or `ERROR` consistently across connectors.
- Add configuration for GCP and PostgreSQL credentials via environment variables, with a clear separation between runtime config and control YAML content.
- Make the batch runner non-interactive and scheduler-compatible so Phase 2 can wrap it in a Cloud Run Job entrypoint without refactoring core execution.

## Public Interfaces
- CLI commands:
  - `run-control --path <control.yaml>`
  - `run-control-set --path <control-set.yaml>`
  - `list-runs`
  - `search-evidence`
- YAML interfaces:
  - Existing control YAML schema remains source-compatible after relocation to `/controls/`
  - New control-set YAML adds grouped execution for scheduled runs
- Storage interfaces:
  - `RawEvidenceStore.write(...) -> raw_evidence_uri`
  - `MetadataStore.write_run(...)`
  - `MetadataStore.write_evidence(...)`
  - `MetadataStore.search(...)`

## Test Plan
- Unit tests for control YAML parsing, schema normalization, registry resolution, evaluator behavior, hash generation, and GCS/Postgres adapter contracts.
- Unit tests covering current YAML patterns from GCP, Jira, and Entra so the loader proves cross-connector compatibility even though only GCP is implemented in Phase 1.
- One end-to-end runner test with mocked GCP client, mocked GCS store, and test Postgres boundary to verify: control load, collection, normalization, evaluation, hash generation, raw evidence write, and metadata write.
- CLI tests for `run-control` and `run-control-set` success/failure behavior and readable operator output.
- Manual verification scenario: execute one bucket-focused GCP control and one control set, confirm raw JSON lands in GCS and metadata is queryable from PostgreSQL.

## Assumptions
- Phase 1 uses real GCS and PostgreSQL abstractions from the start; no SQLite or file-based local fallback is part of this milestone.
- CLI is the only operator interface in this phase; MCP, API, and UI are explicitly later work.
- Scheduler infrastructure is out of scope for implementation planning in this milestone, but the runner must be compatible with future Cloud Run Job execution.
- Bucket/storage controls are the first end-to-end GCP path, but the connector architecture should not hardcode itself to buckets only.
- The workspace currently contains concept docs and YAMLs but no app scaffold, so the implementation will be a clean initial project setup around the `/controls/` catalog.
