# Tasks: mTLS Transport Security for Agent Communication

**Input**: Design documents from `specs/003-mtls-transport-security/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Test tasks are included since this is a controller change requiring unit and integration test coverage per the constitution (Principle II: Spec-Anchored Testing).

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Phase 1: Setup

**Purpose**: CRD type changes, flag defaults, and code generation that all stories depend on

- [ ] T001 Change `mTLSMode` default from `disabled` to `permissive` in `api/v1alpha1/agentruntime_types.go`. Update the kubebuilder default marker on the `MTLSMode` field. Add `ConditionTypeMTLSReady` constant (e.g., `MTLSReady`) to the condition type constants.
- [ ] T002 Run `make generate` and `make manifests` to regenerate deepcopy functions and CRD manifests. Verify `zz_generated.deepcopy.go` is updated and `config/crd/bases/` has the updated AgentRuntime CRD with the new default.
- [ ] T003 [P] Change `--enable-verified-fetch` flag default from `false` to `true` in `cmd/main.go`. This makes SpiffeFetcher the default card fetcher when SPIRE is available.
- [ ] T004 [P] Change `--require-a2a-signature`, `--signature-audit-mode`, and `--enforce-network-policies` flag defaults from `true` to `false` in `cmd/main.go`. Add deprecation warning logs at startup when any of these flags are explicitly set to `true`: `slog.Warn("flag deprecated", "flag", name, "replacement", "mTLS transport security", "removal", "next release")`.

**Checkpoint**: CRD types updated, flags changed, code generated. Ready for controller changes.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core ConfigMap generation and MTLSReady condition logic that all user stories need

**CRITICAL**: No user story work can begin until this phase is complete

- [ ] T005 Add `mtls:` block injection to the authbridge ConfigMap generation in `internal/controller/agentruntime_config.go` (or the relevant ConfigMap construction code in `internal/controller/agentruntime_controller.go`). When `mTLSMode` is `permissive` or `strict`, include `mtls:\n  mode: <value>` in the authbridge config YAML. When `mTLSMode` is `disabled`, omit the `mtls:` block entirely. Ensure the `spiffe:` block is also present when `mtls:` is included. The config hash must change when `mTLSMode` changes.
- [ ] T006 Add `MTLSReady` condition logic to the reconcile loop in `internal/controller/agentruntime_controller.go`. After resolving the target workload, evaluate mTLS readiness: if `mTLSMode` is `disabled`, set `MTLSReady=True` with reason `MTLSDisabled`; if `mTLSMode` is `permissive` or `strict`, check whether spiffe-helper is present in the workload's pod template (via volume mounts or init containers) — if present set `MTLSReady=True` with reason `SPIREAvailable`, if absent set `MTLSReady=False` with reason `SPIREUnavailable` and message `"mTLS requires SPIRE; either deploy SPIRE or set mTLSMode: disabled"`. Follow the save/restore pattern for in-memory status around any Patch calls (Constitution I).

**Checkpoint**: ConfigMap generation and MTLSReady condition ready. User story implementation can begin.

---

## Phase 3: User Story 1 — Agent-to-Agent mTLS by Default (Priority: P1) MVP

**Goal**: Agents deployed with SPIRE communicate over mTLS automatically without explicit mTLSMode configuration because mTLSMode defaults to permissive.

**Independent Test**: Deploy two agent workloads with SPIRE, create AgentRuntimes with no explicit mTLSMode, verify the authbridge ConfigMap contains the `mtls: mode: permissive` block and the workload pods have spiffe-helper and mTLS-configured sidecars.

### Tests for User Story 1

- [ ] T007 [P] [US1] Add unit tests for ConfigMap `mtls:` block generation in `internal/controller/agentruntime_controller_test.go`. Test cases: (a) mTLSMode unset (default permissive) → ConfigMap contains `mtls: mode: permissive`; (b) mTLSMode `strict` → ConfigMap contains `mtls: mode: strict`; (c) mTLSMode `disabled` → ConfigMap has no `mtls:` block; (d) `spiffe:` block is present alongside `mtls:` block. Create objects in envtest and read back the ConfigMap from the API server to verify (Constitution II).
- [ ] T008 [P] [US1] Add unit tests for `MTLSReady` condition in `internal/controller/agentruntime_controller_test.go`. Test cases: (a) mTLSMode permissive + SPIRE available → `MTLSReady=True/SPIREAvailable`; (b) mTLSMode permissive + SPIRE absent → `MTLSReady=False/SPIREUnavailable`; (c) mTLSMode disabled → `MTLSReady=True/MTLSDisabled`; (d) condition transitions when SPIRE becomes available. Read back AgentRuntime from envtest API server to verify conditions (Constitution II).
- [ ] T009 [P] [US1] Add unit test for config hash change on mTLSMode transition in `internal/controller/agentruntime_controller_test.go`. Verify that changing mTLSMode from `permissive` to `strict` or `disabled` results in a different `kagenti.io/config-hash` annotation on the workload pod template.

### Implementation for User Story 1

- [ ] T010 [US1] Wire the ConfigMap `mtls:` block and `MTLSReady` condition into the reconcile flow. Verify that a new AgentRuntime with no explicit mTLSMode gets `permissive` behavior: the authbridge ConfigMap contains the `mtls:` block, the config hash includes it, and the `MTLSReady` condition is set appropriately. Ensure the save/restore pattern is used around metadata patches before Status().Update() (Constitution I, III).
- [ ] T011 [US1] Verify that changing `mTLSMode` on an existing AgentRuntime triggers a rolling restart via config hash change on the workload's pod template annotation.

**Checkpoint**: Agent-to-agent mTLS defaults to permissive. ConfigMap and conditions are correct. Rolling restart on mode change works.

---

## Phase 4: User Story 2 — Controller-to-Agent Communication over mTLS (Priority: P1)

**Goal**: The operator controller uses SpiffeFetcher by default when SPIRE is available. Transport security metadata is recorded in AgentRuntime.status.card.

**Independent Test**: Deploy an agent with SPIRE, create an AgentRuntime, verify `status.card.transportSecurity` is `mtls` and `status.card.attestedAgentSpiffeID` is populated.

### Tests for User Story 2

- [ ] T012 [P] [US2] Add unit tests for SpiffeFetcher as default in `internal/controller/agentruntime_controller_test.go`. Test cases: (a) SPIRE configured (verified-fetch enabled) → SpiffeFetcher used → `status.card.transportSecurity` is `mtls`; (b) SPIRE not configured → DefaultFetcher used → `status.card.transportSecurity` is `http`; (c) `status.card.attestedAgentSpiffeID` populated on mTLS fetch. Use stub fetchers that return controlled data AND read back from envtest API server (Constitution II).
- [ ] T013 [P] [US2] Add unit test for `--enable-verified-fetch` kill switch in `internal/controller/agentruntime_controller_test.go`. Verify that when `EnableVerifiedFetch=false`, the controller falls back to DefaultFetcher even when SPIRE is configured.

### Implementation for User Story 2

- [ ] T014 [US2] Wire SpiffeFetcher as the default fetcher in the reconcile loop in `internal/controller/agentruntime_controller.go`. When `EnableCardDiscovery` is true and `EnableVerifiedFetch` is true (now default) and the SPIRE socket is configured, use `SpiffeFetcher`. Otherwise fall back to `DefaultFetcher`. Ensure `status.card.transportSecurity` and `status.card.attestedAgentSpiffeID` are populated from the fetch result. Follow save/restore pattern for status around Patch calls (Constitution I).

**Checkpoint**: Controller fetches agent cards over mTLS by default. Transport security metadata visible in AgentRuntime status.

---

## Phase 5: User Story 3 — Clear Error When SPIRE Is Unavailable (Priority: P1)

**Goal**: Operators without SPIRE see a clear MTLSReady=False condition with actionable guidance.

**Independent Test**: Create an AgentRuntime in a cluster without SPIRE, verify the MTLSReady condition is False with reason SPIREUnavailable and a helpful message.

### Tests for User Story 3

- [ ] T015 [P] [US3] Add unit test for SPIRE unavailable error condition in `internal/controller/agentruntime_controller_test.go`. Test cases: (a) no spiffe-helper in workload → `MTLSReady=False/SPIREUnavailable` with message containing `"mTLS requires SPIRE"`; (b) mTLSMode disabled → no error, `MTLSReady=True/MTLSDisabled`; (c) SPIRE becomes available → `MTLSReady` transitions to `True`. Read back from envtest (Constitution II).

### Implementation for User Story 3

- [ ] T016 [US3] Verify that the MTLSReady condition logic from T006 correctly handles the SPIRE-unavailable case end-to-end. Ensure the AgentRuntime `Ready` condition reflects MTLSReady — if MTLSReady is False, Ready should also be False (or at least surface the issue). Verify the error message is actionable.

**Checkpoint**: SPIRE-unavailable scenario produces clear, actionable error conditions.

---

## Phase 6: User Story 4 — JWS Signing Pipeline Deprecation Warning (Priority: P2)

**Goal**: Operators using legacy signing flags see deprecation warnings directing them to mTLS.

**Independent Test**: Start the operator with `--require-a2a-signature=true` and verify deprecation warning in logs.

### Tests for User Story 4

- [ ] T017 [P] [US4] Add unit test for deprecation warnings in a test file for `cmd/main.go` logic (or in an existing test that exercises flag parsing). Verify that when `--require-a2a-signature=true` is set, a deprecation warning is logged. Verify no warning when the flag is at its new default (`false`).

### Implementation for User Story 4

- [ ] T018 [US4] Verify the deprecation warning logic from T004 works correctly. Test that all three deprecated flags (`--require-a2a-signature`, `--signature-audit-mode`, `--enforce-network-policies`) emit structured warnings when set to `true`. Ensure no warnings when flags are at defaults.

**Checkpoint**: Legacy signing flags deprecated with clear warnings.

---

## Phase 7: Authbridge Verification (kagenti-extensions)

**Purpose**: Verify the authbridge mTLS implementation is complete and consistent across all proxy modes

- [ ] T019 [P] Verify `authbridge/cmd/authbridge-envoy/main.go` has the same mTLS wiring as proxy and lite modes. Check whether envoy mode uses native Envoy TLS contexts or the Go-level `authtls` package. Document findings.
- [ ] T020 [P] Verify the `cfg.MTLS` config schema in `authbridge/authlib/config/config.go` matches the `mtls: mode:` block the operator will generate (from contracts/authbridge-mtls-config.md).
- [ ] T021 Add mTLS integration tests in `authbridge/tests/` (or appropriate test location). Test cases: (a) permissive mode accepts both TLS and plaintext inbound; (b) strict mode rejects plaintext inbound; (c) certificate rotation mid-session (spiffe-helper writes new SVID, next handshake uses it).

**Checkpoint**: Authbridge mTLS verified across all modes. Config schema matches operator expectations.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: E2E test, documentation, cleanup

- [ ] T022 [P] Add an e2e test scenario in `test/e2e/` that deploys two agents with SPIRE, creates AgentRuntimes with default mTLSMode, and verifies: (a) authbridge ConfigMaps contain `mtls:` block; (b) MTLSReady conditions are True; (c) controller fetches cards over mTLS (status.card.transportSecurity is mtls).
- [ ] T023 [P] Update `GETTING_STARTED.md` or operator documentation to reflect mTLS-by-default behavior, the `mTLSMode` field, and how to opt out with `mTLSMode: disabled`.
- [ ] T024 Run `make generate && make manifests && make test` to verify all changes pass existing tests and no regressions.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001, T002 for CRD types)
- **User Stories (Phase 3-6)**: All depend on Foundational phase completion
  - US1, US2, US3 are all P1 and can proceed in parallel after Phase 2
  - US4 (P2) can proceed independently after Phase 1 (flag changes in T004)
- **Authbridge Verification (Phase 7)**: Independent — can run in parallel with any phase
- **Polish (Phase 8)**: Depends on all user stories being complete

### User Story Dependencies

- **US1 (Agent-to-Agent mTLS)**: Depends on Phase 2 (ConfigMap + MTLSReady) — no other story deps
- **US2 (Controller-to-Agent mTLS)**: Depends on Phase 2 — independent of US1
- **US3 (SPIRE Unavailable Error)**: Depends on Phase 2 (MTLSReady condition) — independent of US1/US2
- **US4 (Deprecation Warnings)**: Depends only on T004 (flag defaults) — can start early

### Within Each User Story

- Tests written first, verify they reference the right conditions
- Implementation wires the behavior
- Read back from envtest API server to verify (Constitution II)

### Parallel Opportunities

- T003 and T004 can run in parallel (different concerns in cmd/main.go)
- T007, T008, T009 can all run in parallel (different test cases, same file but independent sections)
- T012, T013 can run in parallel
- T019, T020 can run in parallel (different files in kagenti-extensions)
- T022, T023 can run in parallel (e2e test vs documentation)
- Phase 7 (authbridge verification) can run entirely in parallel with operator work

---

## Parallel Example: User Story 1

```bash
# Launch all tests for User Story 1 together:
Task: "Unit tests for ConfigMap mtls block in agentruntime_controller_test.go"
Task: "Unit tests for MTLSReady condition in agentruntime_controller_test.go"
Task: "Unit test for config hash change in agentruntime_controller_test.go"

# After tests defined, implement:
Task: "Wire ConfigMap and MTLSReady into reconcile flow"
Task: "Verify rolling restart on mTLSMode change"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (CRD defaults, flag changes)
2. Complete Phase 2: Foundational (ConfigMap mtls block, MTLSReady condition)
3. Complete Phase 3: User Story 1 (agent-to-agent mTLS by default)
4. **STOP and VALIDATE**: Test with two agents + SPIRE in a cluster
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → mTLS infrastructure ready
2. Add US1 → Agent-to-agent mTLS works → Test independently (MVP!)
3. Add US2 → Controller fetches over mTLS → Test independently
4. Add US3 → Clear errors without SPIRE → Test independently
5. Add US4 → Deprecation warnings → Test independently
6. Authbridge verification + E2E → Confidence across all modes
