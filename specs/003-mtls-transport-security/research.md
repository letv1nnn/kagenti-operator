# Research: mTLS Transport Security for Agent Communication

## R1: Authbridge MTLS Config Schema

**Decision**: The authbridge sidecar reads mTLS configuration from a top-level `mtls:` block in its YAML config file. The operator must inject this block into the authbridge ConfigMap.

**Schema** (from `authbridge/authlib/config/config.go`):
```yaml
mtls:
  mode: permissive  # or strict
```

**Resolved mode behavior**:
- `permissive`: Inbound uses byte-peek TLS-sniffing (accepts both TLS and plaintext). Outbound uses plaintext.
- `strict`: Inbound rejects non-TLS connections. Outbound requires TLS.

**Rationale**: The config schema is already defined in authbridge. The operator just needs to populate it based on `AgentRuntime.spec.mTLSMode`.

**Alternatives considered**: None — the schema is fixed by the authbridge implementation.

## R2: Authbridge SPIFFE Config Schema

**Decision**: The authbridge sidecar reads SPIFFE configuration from a `spiffe:` block. This is already injected by the operator when spiffe-helper is present.

**Schema**:
```yaml
spiffe:
  socket: "unix:///spiffe-workload-api/spire-agent.sock"
  mirrorFiles: true
  mirrorDir: "/spiffe-certs"
```

**Rationale**: The SPIFFE block is a prerequisite for `mtls:`. The operator already handles this — no changes needed.

## R3: ConfigMap Hash and Rolling Restart

**Decision**: When `mTLSMode` changes on an AgentRuntime, the ConfigMap hash annotation on the workload's pod template must change, triggering a rolling restart. The existing `kagenti.io/config-hash` annotation already handles this for the 3-layer (now 2-layer) config merge.

**Implementation note**: Adding the `mtls:` block to the ConfigMap content changes the hash automatically — no special handling needed beyond ensuring the `mtls:` block is included in the config that gets hashed.

**Rationale**: Reuses existing config-hash mechanism. No new infrastructure.

## R4: SpiffeFetcher Default Behavior

**Decision**: When the controller pod has the SPIRE Workload API socket available (`--verified-fetch-spiffe-socket`), use `SpiffeFetcher` as the primary fetcher. Fall back to `DefaultFetcher` only when SPIRE is not configured.

**Current behavior**: `SpiffeFetcher` is only used when `--enable-verified-fetch=true` (default: false). Changing the default to `true` and keeping the flag as a kill switch satisfies the "enabled by default, disabled is opt-in" requirement.

**Rationale**: Minimal code change — just a default value flip. The flag remains for emergency rollback per Constitution V.

## R5: MTLSReady Condition Design

**Decision**: Add a new condition type `MTLSReady` to AgentRuntime status conditions.

**Condition states**:

| Reason | Status | When |
|--------|--------|------|
| `SPIREAvailable` | True | SPIRE is deployed and certificates are available |
| `SPIREUnavailable` | False | mTLSMode is non-disabled but SPIRE is not deployed or unreachable |
| `MTLSDisabled` | True | mTLSMode is explicitly set to `disabled` (no mTLS expected) |

**Detection**: The controller checks whether the spiffe-helper init container is present in the workload's pod template and whether the SPIRE agent socket volume mount exists. If mTLSMode is `permissive` or `strict` but these are absent, `MTLSReady=False`.

**Rationale**: Follows the existing condition pattern (TargetResolved, ConfigResolved, Ready). Uses the same `metav1.Condition` type.

## R6: Deprecation Warning Implementation

**Decision**: At operator startup in `cmd/main.go`, after flag parsing, check if any legacy signing flags are set to `true` and log structured deprecation warnings.

**Flags to deprecate**:
- `--require-a2a-signature` (default: false)
- `--signature-audit-mode` (default: false)
- `--enforce-network-policies` (default: false)

**Warning format**: `slog.Warn("flag deprecated", "flag", name, "replacement", "mTLS transport security", "removal", "next release")`

**Rationale**: Structured logging matches existing operator patterns. No runtime behavior change — just warnings.

## R7: Authbridge-Envoy mTLS Status

**Decision**: Verify whether `authbridge-envoy/main.go` has the same mTLS wiring as proxy and lite modes.

**Finding**: Need to verify — the envoy mode uses `ext_proc`/`ext_authz` listeners instead of HTTP reverse/forward proxy. mTLS in envoy mode may be handled via Envoy's native `DownstreamTlsContext`/`UpstreamTlsContext` rather than the Go-level `authtls` package. The operator generates envoy bootstrap config, so TLS contexts need to be included there.

**Action**: Verify during implementation. If envoy mode uses native Envoy TLS, the operator must inject TLS context config into the envoy bootstrap ConfigMap.
