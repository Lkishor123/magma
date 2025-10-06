# Domain Proxy Implementation Status vs. SAS Product Plan

This document summarizes which items from the provided SAS product plan are already implemented under `dp/` and which items would require net-new work.

## Legend

* **Status**
  * ✅ Implemented in the current codebase
  * ⚠️ Partially implemented or limited
  * ⛔️ Not implemented yet

---

## Phase 0 — “Foundations First”

| Plan item | Status | Evidence | Notes |
| --- | --- | --- | --- |
| Mutual TLS with certificate revocation checking | ✅ | `RequestRouter` enforces mTLS (client cert + CA verification) and optionally validates CRLs before sending SAS messages.【F:dp/cloud/python/magma/configuration_controller/request_router/request_router.py†L31-L93】 | Covers TLS handshake and CRL checks when configured. No explicit CBRS CP schema validation beyond trust anchors.
| Machine-readable Regulatory Profile | ⛔️ | No references to "regulatory profile" or similar persisted policy constructs exist in `dp/` (search returns no matches).【8f7aea†L1-L1】 | Would require new data model and config pipeline.

## Phase 1 — “Baseline SAS↔CBSD”

| Plan item | Status | Evidence | Notes |
| --- | --- | --- | --- |
| Registration flow | ✅ | SAS request mapping includes `registrationRequest`; responses drive CBSD state updates and ID assignment.【F:dp/cloud/python/magma/mappings/request_mapping.py†L14-L21】【F:dp/cloud/python/magma/configuration_controller/response_processor/strategies/response_processing.py†L66-L107】 | Active mode controller auto-generates registration when desired state requires it.【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/message_generator.go†L34-L79】
| Spectrum Inquiry | ✅ | Request mapping plus response handler creates available channel inventory for each CBSD.【F:dp/cloud/python/magma/mappings/request_mapping.py†L14-L21】【F:dp/cloud/python/magma/configuration_controller/response_processor/strategies/response_processing.py†L110-L147】 | Channel data later feeds grant calculations.【F:dp/cloud/python/magma/radio_controller/services/active_mode_controller/service.py†L300-L384】
| Grant issuance | ✅ | Grant processor calculates compliant EIRP per channel selection, and responses persist grant state, heartbeat intervals, and timing.【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/sas/grant.go†L20-L45】【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/sas/eirp/calculator.go†L24-L105】【F:dp/cloud/python/magma/configuration_controller/response_processor/strategies/response_processing.py†L149-L181】
| Heartbeat maintenance | ✅ | Heartbeat requests/responses update grant authorization state and track transmit expiry metadata.【F:dp/cloud/python/magma/configuration_controller/response_processor/strategies/response_processing.py†L183-L219】【F:dp/cloud/python/magma/db_service/models.py†L110-L159】 | Active mode controller schedules heartbeat traffic when grants are active.【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/message_generator.go†L34-L79】
| Relinquishment & Deregistration | ✅ | Generator and handlers issue relinquishment/deregistration requests and clear associated database state.【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/message_generator.go†L52-L78】【F:dp/cloud/python/magma/configuration_controller/response_processor/strategies/response_processing.py†L222-L258】

## Phase 2 — “Decisioning & Protections”

| Plan item | Status | Evidence | Notes |
| --- | --- | --- | --- |
| Grant engine honors technical limits (channels/EIRP) | ✅ | Grant builder derives EIRP bounds from CBSD capabilities, antenna gain, and available channel masks before issuing requests.【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/sas/grant.go†L20-L45】【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/sas/eirp/calculator.go†L24-L105】 | Stored CBSD model keeps per-device capabilities and preferred frequencies to drive policy.【F:dp/cloud/python/magma/db_service/models.py†L110-L220】
| Automatic pre-emption of GAA / revocation on protections | ⚠️ | Active mode controller can mark grants for relinquishment when devices go inactive or operator requests it, but no explicit incumbent/ESC trigger pipeline exists.【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/message_generator.go†L44-L78】 | Protection-trigger ingestion (e.g., from ESC or PAL coexistence) would still need to be wired in.
| ESC event handling | ⛔️ | Only tooling stubs under `dp/tools/fake_sas` mention ESC flows; service packages contain no ESC ingestion logic.【5d1067†L1-L11】 | Implementing real ESC integration remains outstanding.

## Phase 3 — “Federation & R2 Features”

| Plan item | Status | Evidence | Notes |
| --- | --- | --- | --- |
| SAS↔SAS coordination | ⛔️ | No SAS-SAS protocol implementation is present; only fake SAS tooling references the spec.【5d1067†L1-L11】 | Would require new services for WINNF-TS-0096/3010 procedures.
| Release-2 feature advertisement/extensions | ⛔️ | No feature-capability plumbing or WINNF-TS-3002/3003 identifiers found in `dp/` code (search for "feature" yields nothing).【ef7072†L1-L1】 | Future work to expose optional R2 behaviors.

## Phase 4 — “Operations-Grade”

| Plan item | Status | Evidence | Notes |
| --- | --- | --- | --- |
| Observability & auditing | ✅ | Domain Proxy logs each CBSD interaction to an external consumer and exposes Prometheus metrics for request/response processing.【F:dp/cloud/go/services/dp/servicers/cbsd_manager.go†L34-L170】【F:dp/cloud/go/services/dp/logs_pusher/logs_pusher.go†L1-L33】【F:dp/cloud/python/magma/configuration_controller/metrics.py†L16-L30】 | Provides audit trail and performance telemetry hooks.
| Availability / DR / scale automation | ⛔️ | No code or configuration referencing multi-AZ deployments, failover orchestration, or chaos testing is present (search for related terms is empty).【3cf100†L1-L1】 | Would require architectural work outside current scope.
| Certification harness integration | ⚠️ | Helm values allow deploying optional certification helpers, but no automated harness runner is implemented in services.【F:dp/README.md†L44-L54】 | Additional tooling needed to execute WInnForum certification suites end-to-end.

## Phase 5 — “O-RAN Integration (Optional)”

| Plan item | Status | Evidence | Notes |
| --- | --- | --- | --- |
| Domain Proxy mediation for CBSD fleets | ✅ | Domain Proxy service stack (gRPC servicer, database, active mode controller) manages CBSD state, grant lifecycle, and orchestration actions for large fleets.【F:dp/cloud/go/services/dp/servicers/cbsd_manager.go†L34-L200】【F:dp/cloud/go/services/dp/active_mode_controller/action_generator/message_generator.go†L34-L79】【F:dp/cloud/python/magma/radio_controller/services/active_mode_controller/service.py†L300-L384】 | Supports carrier aggregation, grant redundancy, and multi-grant scheduling.
| RAN/SMO M-Plane hooks | ⛔️ | No NETCONF/YANG or SMO integration code present under `dp/` (search for NETCONF/M-plane artifacts yields none).【d07182†L1-L1】【8516bf†L1-L1】 | Integration with O-RAN management stacks still pending.

---

### Summary

Most baseline SAS↔CBSD interactions, grant management, and Domain Proxy orchestration exist today. Gaps include regulatory profiling, incumbent/ESC driven protections, SAS↔SAS federation, Release-2 feature advertisement, disaster-recovery automation, certification harness execution, and O-RAN SMO control-plane integrations.
