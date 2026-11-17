# External governance integration boundary

## Purpose

This document records the supported integration boundary for an external enterprise governance service. It deliberately keeps tenant, organization, identity-mapping, entitlement, approval, and proprietary policy business logic outside this public repository.

The integration must preserve the native Agentgateway data plane and its existing configuration, routing, model-provider, MCP, A2A, security, telemetry, and administration features.

## Native integration surfaces

### HTTP and LLM authorization

Use the native `AgentgatewayPolicy` traffic `extAuth` integration for synchronous authorization decisions. The policy supports gRPC and HTTP backends, request metadata derived from verified gateway context/CEL, optional request-body forwarding, response metadata, and an explicit failure mode.

For mandatory authorization, configure the failure mode explicitly as fail-closed and test both an allow decision and every denial/error path. Do not treat client-supplied identity headers as authoritative. The external service must validate signed identity evidence or a mutually authenticated service channel before deriving tenant or user scope.

### LLM prompt and response guardrails

Use the native backend `ai.promptGuard` webhook integration for external request- and response-phase policy evaluation. Configure both phases where the policy requires them. The webhook action and failure mode must be explicit; mandatory policy evaluation must reject on service failure.

Do not assume that streaming response inspection is equivalent to whole-response inspection. The current streaming guardrail implementation evaluates buffered windows with a bounded overlap. Text already emitted from an earlier passing window cannot be recalled, and full-response context or patterns longer than the overlap may be missed. Critical controls requiring complete-response context need an execution path that buffers the complete response or an independently validated enforcement design.

### MCP tool governance

Use native `backend.mcp.guardrails.processors` with a remote processor for tool-call and tool-result checks. The processor supports method scoping, CEL metadata, header allow/disallow lists, and explicit failure mode. Configure mandatory operations with `FailClosed`.

Avoid forwarding credentials or unrelated headers to the policy service. Prefer a strict allow-list and explicitly disallow `authorization` unless a reviewed protocol contract requires it. A metadata expression such as `jwt.sub` is only trustworthy when the JWT has already been validated by the gateway.

### Native policy and budget capabilities

Agentgateway already provides native CEL-based authorization/policy facilities, LLM guardrails, MCP guardrails, request logging, telemetry, and API-key budget enforcement. Preserve these capabilities rather than replacing them.

The existing budget model is attached to an API key. Do not represent it as a complete organization/project/department/team/user budget hierarchy without an external authoritative policy and accounting layer.

## External service responsibilities

The external service is authoritative for:

- Tenant and organization membership resolution.
- Project, department, team, and user entitlements.
- Effective policy composition across scopes.
- Approval, policy versioning, simulation, rollback, and immutable audit history.
- Directory synchronization and identity-to-application mapping.
- Hierarchical quotas and chargeback rules.
- Any proprietary business rules.

The data plane must receive a deterministic decision with a stable decision identifier, policy version, outcome, and a bounded reason code. Do not place secrets, raw credentials, or unnecessary prompt content in decision metadata or audit logs.

## Security invariants

1. Fail closed for mandatory authorization and guardrail checks.
2. Authenticate the external policy service using TLS and an approved service identity.
3. Treat all caller-supplied tenant/user headers as untrusted until cryptographically bound to verified identity.
4. Minimize forwarded headers and request bodies.
5. Never log provider keys, directory passwords, session cookies, or bearer tokens.
6. Bound timeouts, request sizes, concurrency, retries, and connection pools.
7. Make policy version and decision ID available for audit correlation.
8. Do not turn an unavailable policy service into an allow decision.
9. Keep proprietary policy implementation and secrets out of this public fork.

## Required verification before production

- Valid authenticated request is allowed only for the intended tenant and scope.
- Missing, expired, malformed, wrong-issuer, and wrong-audience credentials are denied.
- Cross-tenant identifier substitution is denied.
- Explicit deny wins over any lower-scope allow.
- A lower scope cannot weaken a mandatory global restriction.
- Policy-service timeout, TLS failure, malformed response, and 5xx all fail closed for mandatory controls.
- MCP request and response phases are tested independently.
- LLM prompt and response phases are tested independently.
- Streaming limitations are covered by documented test cases.
- Header allow-lists are tested for credential leakage.
- Usage, cost, and token values are recorded only when the source reports them reliably.
- Native upstream tests and feature behavior remain unchanged.

## Source locations

- `controller/pkg/agentgateway/plugins/testdata/trafficpolicy/extauthz-grpc.yaml`
- `controller/pkg/agentgateway/plugins/testdata/trafficpolicy/extauthz-http.yaml`
- `controller/pkg/agentgateway/plugins/testdata/backendpolicy/ai-guardrails-audit.yaml`
- `controller/pkg/agentgateway/plugins/testdata/backendpolicy/guardrails.yaml`
- `crates/agentgateway/src/mcp/guardrails/mod.rs`
- `crates/agentgateway/src/llm/policy/streaming_guardrails.rs`
- `crates/agentgateway/src/http/budget/mod.rs`
- `crates/agentgateway/src/telemetry/log_store.rs`

This document is an integration contract, not evidence that an external governance service has been implemented or deployed.
