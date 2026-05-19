# Custom TLS Configuration for Broker Client Connections

**Issue:** [#659](https://github.com/Kuadrant/mcp-gateway/issues/659)
**Date:** 2026-05-19
**Status:** Draft

## Problem

The MCP broker connects to upstream MCP servers using Go's default HTTP transport, which only trusts publicly-trusted Certificate Authorities (CAs). In-cluster MCP servers often use private CAs (OpenShift service-serving CA, cert-manager with a private issuer, self-signed certificates), causing the broker to reject connections with certificate verification errors. There is currently no way to provide a custom CA bundle for the broker to trust when connecting to these backends.

This only affects the broker's direct connections to upstream MCP servers (tool/list discovery, initialization, session management). The tool/call path flows through Envoy, which has its own TLS handling.

## Solution

Add per-backend custom CA certificate support via a `caCertSecretRef` field on `MCPServerRegistration`. This follows the same pattern as the existing `credentialRef` field: a reference to a Kubernetes Secret containing the CA certificate PEM data.

When a backend has a `caCertSecretRef` configured, the broker creates a custom HTTP client that trusts both the system CA pool and the additional CA certificate. Backends without `caCertSecretRef` continue using Go's default trust store — fully backwards compatible.

### User-facing API

```yaml
apiVersion: mcp.kuadrant.io/v1alpha1
kind: MCPServerRegistration
metadata:
  name: my-private-server
  namespace: mcp-gateway
spec:
  targetRef:
    name: my-server-route
  prefix: private
  caCertSecretRef:
    name: my-ca-bundle       # Secret containing the CA certificate
    key: ca.crt              # optional, defaults to "ca.crt"
```

The referenced Secret must:
- Exist in the same namespace as the MCPServerRegistration
- Carry the label `mcp.kuadrant.io/secret: "true"` (same requirement as `credentialRef`)
- Contain a PEM-encoded CA certificate at the specified key

Example Secret:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-ca-bundle
  namespace: mcp-gateway
  labels:
    mcp.kuadrant.io/secret: "true"
type: Opaque
data:
  ca.crt: <base64-encoded PEM certificate>
```

### Design decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| `insecureSkipVerify` | Not included | Team consensus: providing a CA bundle is sufficient for all use cases. Skip-verify encourages bad practices. |
| `serverName` override | Not included (defer) | Niche use case (SNI routing, IP-only endpoints). Can be added later if needed. |
| Default key | `ca.crt` | Kubernetes convention for CA bundles. Matches cert-manager and OpenShift service-serving CA output. |
| Trust model | Append to system pool | Custom CA is added alongside system CAs, not replacing them. Backends with public certs continue to work. |
| Scope | Broker path only | Tool/call requests flow through Envoy. This feature covers broker discovery/init connections only. |

## Architecture

### Data flow

```
MCPServerRegistration (CRD)
  └── caCertSecretRef → Kubernetes Secret (PEM data)
        │
        ▼
Controller (mcpserverregistration_controller.go)
  └── buildMCPServerConfig()
        │  reads Secret, extracts PEM string
        ▼
config.MCPServer struct
  └── CACert field (PEM string)
        │
        ▼
Config Secret (YAML serialization)
  └── config.yaml: servers[].caCert: "<PEM>"
        │
        ▼
Broker runtime (LoadConfig)
  └── config.MCPServer.CACert
        │
        ▼
Upstream manager (mcp.go)
  └── Connect()
        ├── CACert empty? → use default HTTP client (today's behavior)
        └── CACert present? → create custom HTTP client:
              1. x509.SystemCertPool()        — load system CAs
              2. pool.AppendCertsFromPEM(...)  — add custom CA
              3. &http.Client{Transport: &http.Transport{
                   TLSClientConfig: &tls.Config{RootCAs: pool}
                 }}
              4. transport.WithHTTPBasicClient(client) — pass to mcp-go
```

### Components modified

| Component | File(s) | Change |
|-----------|---------|--------|
| CRD types | `api/v1alpha1/types.go` | Add `CACertSecretRef *SecretReference` to `MCPServerRegistrationSpec` |
| CRD defaults | `api/v1alpha1/defaults.go` | Set default key to `ca.crt` for `CACertSecretRef` |
| Generated CRDs | `config/crd/`, `charts/`, `bundle/` | Regenerated via `make generate-all` |
| Controller | `internal/controller/mcpserverregistration_controller.go` | Read CA cert from Secret in `buildMCPServerConfig()`; update `findMCPServerRegistrationsForSecret()` to match `caCertSecretRef` |
| Config types | `internal/config/types.go` | Add `CACert string` field to `MCPServer` |
| Upstream manager | `internal/broker/upstream/mcp.go` | Create custom HTTP client with CA cert in `Connect()` |
| API reference docs | `docs/reference/mcpserverregistration.md` | Document `caCertSecretRef` field |

### Components NOT modified

| Component | Why |
|-----------|-----|
| Config writer (`config_writer.go`) | YAML serialization handles the new `CACert` string field automatically |
| Broker main (`cmd/mcp-broker-router/main.go`) | Config flow is unchanged; the new field flows through existing Viper unmarshaling |
| Router (`internal/mcp-router/`) | Router handles tool/call requests which flow through Envoy, not the broker's HTTP client |
| Secret watcher setup | The controller already watches Secrets with the `mcp.kuadrant.io/secret=true` label; `caCertSecretRef` secrets use the same label, so the existing watcher covers them |

## Secret rotation

When a CA certificate Secret is updated:

1. The controller's existing Secret watcher detects the change (predicate: `mcp.kuadrant.io/secret=true` label)
2. `findMCPServerRegistrationsForSecret()` finds all MCPServerRegistrations that reference this Secret (via either `credentialRef` or `caCertSecretRef`)
3. The controller re-reconciles those MCPServerRegistrations, re-reading the Secret and updating the config Secret
4. The broker observes the config change, and recreates upstream connections with the updated CA certificate

No new watcher infrastructure is needed. The existing `findMCPServerRegistrationsForSecret()` function needs to be updated to also check `caCertSecretRef` references (currently it only checks `credentialRef`).

## Error handling

| Error condition | Behavior |
|----------------|----------|
| Secret not found | Status condition on MCPServerRegistration: `Ready=False`, reason `CACertSecretNotFound` |
| Secret missing required label | Status condition: `Ready=False`, reason `CACertSecretMissingLabel` |
| Key not found in Secret | Status condition: `Ready=False`, reason `CACertKeyNotFound` |
| PEM parsing fails (upstream) | Upstream manager logs error, does not connect to backend. Broker continues operating for other backends. |
| System cert pool unavailable | Fall back to an empty pool (append custom CA only). This is a rare edge case on minimal container images. |

## Testing

### Unit tests

- `buildMCPServerConfig()`: with/without `caCertSecretRef`, missing Secret, missing key, missing label, empty PEM data
- `findMCPServerRegistrationsForSecret()`: returns registrations referencing the Secret via `caCertSecretRef`
- Upstream `Connect()` with custom CA cert against a test TLS server using a self-signed cert
- Default key behavior: verify `ca.crt` is used when `Key` is not specified

### Controller integration tests (envtest)

- MCPServerRegistration with `caCertSecretRef` reconciles successfully
- CA Secret update triggers re-reconciliation
- Status conditions set correctly for error cases

### Documentation

- Update `docs/reference/mcpserverregistration.md` with `caCertSecretRef` field documentation
- Consider a guide in `docs/guides/` for using private CAs (can be a follow-up)

## Out of scope

- `insecureSkipVerify`: decided against by team
- `serverName` override: deferred, niche use case
- Client certificate authentication (mTLS): separate feature if needed
- Envoy/router TLS configuration: handled by Gateway API / Istio, not this feature
- E2E tests with actual private CA: needs cluster infrastructure, can be a follow-up PR
