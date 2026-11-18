# MCF AI-KU integration

This directory contains the MCF overlay for the Agentgateway fork.

Native Agentgateway remains the execution plane. MCF AI-KU adds:

- MCF-issued RS256 identity verification through JWKS.
- Backend external authorization for tenant/project/department/team governance.
- Fail-closed guardrail webhooks.
- MCF word controls: mask, block and warn.
- Published guardrail lifecycle controlled by MCF.
- Strict non-streaming response inspection for routes where complete-response policy enforcement is mandatory.

## Security boundary

The MCF governance API is private and must not be exposed as a public endpoint.

Agentgateway authenticates to AI-KU with the MCF governance service credential. The end-user JWT remains the identity credential used by the authorization service.

Project, department and team scope is carried in explicit MCF scope headers. AI-KU verifies that the authenticated user is actually a member of the requested scope before any policy or budget decision is accepted.

## Streaming guardrail decision

Agentgateway's streaming guardrail path evaluates bounded windows. A window cannot retroactively inspect text already emitted to the client.

Therefore MCF's strict full-response guardrail mode uses the buffered/non-streaming response path. Native streaming remains available when the effective MCF policy does not require complete-response inspection.

This is a deliberate security policy, not a silent regression.

## Deployment

Use deploy/mcf-ai-ku/config.yaml as a template only. Replace:

- AI-KU HTTPS hostname.
- AI-KU CA trust path.
- MCF governance API key path.
- Actual model/provider configuration.

Do not commit production credentials.

## Verification

Before release, validate the rendered Agentgateway configuration against the exact Agentgateway binary version deployed by MCF and run the native Agentgateway test suite.
