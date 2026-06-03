# Contract: Authbridge mTLS Configuration

## Owner

kagenti-operator (producer) → authbridge sidecar (consumer)

## Interface

The operator writes an authbridge YAML config into a ConfigMap. The authbridge sidecar reads this config at startup and on hot-reload.

## mTLS Config Block

When `AgentRuntime.spec.mTLSMode` is `permissive` or `strict`, the operator includes:

```yaml
mtls:
  mode: permissive  # or strict
```

When `AgentRuntime.spec.mTLSMode` is `disabled` (or empty before this change), the `mtls:` block is omitted.

## SPIFFE Config Block (existing, no change)

The SPIFFE block is already injected when spiffe-helper is present:

```yaml
spiffe:
  socket: "unix:///spiffe-workload-api/spire-agent.sock"
  mirrorFiles: true
  mirrorDir: "/spiffe-certs"
```

## Behavior Contract

| mTLSMode | mtls block | spiffe block | Inbound behavior | Outbound behavior |
|----------|-----------|-------------|-----------------|------------------|
| `disabled` | omitted | may be present | Plaintext only | Plaintext only |
| `permissive` | `mode: permissive` | required | TLS-sniff: accepts TLS and plaintext | Plaintext |
| `strict` | `mode: strict` | required | TLS required, rejects plaintext | TLS required |

## Validation

- If `mtls:` is present but `spiffe:` is absent, authbridge fails at startup with: `"mtls requires the spiffe block to be configured"`
- The operator MUST ensure both blocks are present when mTLSMode is non-disabled
