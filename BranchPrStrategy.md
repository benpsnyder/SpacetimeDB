# Branch And PR Strategy For The SpacetimeDB Gateway/Auth Work

## Goal

This plan breaks the work described in `Enhancements.md` into separate branches and pull requests that can be reviewed independently by SpacetimeDB maintainers. The strategy keeps the public framing generic, avoids private product names, and separates docs, examples, SDK improvements, and CLI/tooling.

## Ground Rules

- Keep one maintainer-review topic per PR.
- Start with docs that ask for and capture maintainer guidance before adding larger examples or SDK changes.
- Keep Better Auth examples explicit but optional. The general pattern should remain "app-owned auth broker plus SpacetimeDB JWT".
- Avoid coupling the generic server gateway example to Effect, TanStack Start, or Analog. Add framework-specific variants in separate PRs.
- Use generic SaaS terminology: tenant, organization, actor, robot, gateway, customer portal, operator dashboard.
- Keep private names, private paths, and private domain specifics out of every branch.

## Suggested PR Sequence

### PR 1: Public Architecture Issue Draft

Branch: `docs/server-gateway-auth-issue`

Purpose: Add or publish the maintainer-facing issue content from `Enhancements.md`.

Scope:

- Server-side realtime gateway architecture.
- App-owned auth and Better Auth context.
- WSS from server to SpacetimeDB and SSE from server to browser.
- Actor attribution questions.
- Maintainer open questions.

Acceptance criteria:

- The issue is linkable without private names.
- Maintainers can answer the topology and SDK-support questions before implementation work begins.

### PR 2: Server-Side TypeScript SDK Lifecycle Docs

Branch: `docs/server-side-ts-sdk-lifecycle`

Purpose: Document how to use the generated TypeScript SDK from Node/server runtimes.

Scope:

- Supported Node versions and WebSocket runtime expectations.
- Long-lived `DbConnection` lifecycle.
- Reconnect, resubscribe, and rehydrate behavior.
- Reducer calls while subscriptions are active.
- Shutdown behavior for containers and long-running server processes.
- Known limits for serverless environments.

Depends on: PR 1 maintainer feedback.

Acceptance criteria:

- A server-side team can decide whether the TypeScript SDK is suitable for a gateway process.
- The docs state what is supported, what is experimental, and what is not recommended.

### PR 3: Gateway Connection Topologies And Actor Attribution

Branch: `docs/gateway-actor-attribution`

Purpose: Document how a server-side gateway should preserve or model user identity.

Scope:

- Per-user connection topology.
- Service/robot connection topology.
- Hybrid topology.
- `ctx.sender` implications.
- Delegated actor claims and reducer arguments.
- Audit trail recommendations.
- Subscription filtering implications.

Depends on: PR 1 maintainer feedback.

Acceptance criteria:

- The docs clearly explain which topology SpacetimeDB recommends for common SaaS cases.
- The security risks of trusting client-supplied actor IDs are explicit.

### PR 4: App-Owned Auth Broker Guide

Branch: `docs/app-owned-auth-broker`

Purpose: Add a general guide for translating an app session into a short-lived SpacetimeDB JWT.

Scope:

- Broker flow.
- Claim contract.
- JWKS and issuer setup.
- Token lifetime guidance.
- Audience validation.
- Reducer-side claim checks.
- Refresh and reconnect behavior.

Depends on: PR 2 and PR 3.

Acceptance criteria:

- The guide works for Better Auth, Auth.js, custom sessions, or another app-owned session system.
- The guide makes clear that the SpacetimeDB token is not the browser session.

### PR 5: Better Auth Integration Guide

Branch: `docs/better-auth-integration`

Purpose: Add a Better Auth-specific guide that builds on the generic auth broker guide.

Scope:

- Better Auth session plus custom SpacetimeDB JWT broker mode.
- Better Auth OAuth/OIDC provider mode.
- JWKS exposure.
- `resource` audience requirement for JWT access tokens.
- Opaque token warning.
- Better Auth JWT algorithm notes.
- Organization claim projection.
- API-key package fit for robot credentials.

Depends on: PR 4.

Acceptance criteria:

- A Better Auth app can choose between a custom broker endpoint and OAuth/OIDC provider mode.
- The guide explicitly warns that opaque Better Auth access tokens cannot be directly validated by SpacetimeDB via JWKS.

### PR 6: Keycloak/OIDC Identity Migration Guide

Branch: `docs/oidc-identity-migration`

Purpose: Help teams move from an existing OIDC issuer to app-owned auth without losing identity continuity.

Scope:

- `iss` plus `sub` identity derivation.
- Why issuer changes create new SpacetimeDB identities.
- Identity-link tables.
- Dual-issuer migration period.
- Cutover checklist.
- Audit and rollback considerations.

Depends on: PR 4.

Acceptance criteria:

- A team can plan a migration from Keycloak, Auth0, Clerk, or another OIDC provider without binding app authorization directly to raw SpacetimeDB identity alone.

### PR 7: Server Gateway Plus SSE Example

Branch: `examples/server-gateway-sse`

Purpose: Provide an official minimal example of server-side SpacetimeDB WSS and browser SSE relay.

Scope:

- Generated TypeScript SDK used from the server.
- Initial snapshot route or loader.
- SSE endpoint using `EventSource`.
- Subscription updates relayed as typed event envelopes.
- Browser reconnect with `Last-Event-ID`.
- Reducer calls through validated server endpoints.
- Authorization narrowing or stream revocation.
- Graceful shutdown.

Depends on: PR 2, PR 3, and PR 4.

Acceptance criteria:

- The example can run locally.
- It demonstrates one explicit actor-attribution topology.
- It does not require Effect or Better Auth, but leaves clear extension points for both.

### PR 8: TanStack Start Variant

Branch: `examples/tanstack-start-gateway-sse`

Purpose: Add a modern full-stack React variant using TanStack Start conventions.

Scope:

- Server functions for mutations.
- API route for SSE.
- Server-only gateway module.
- Shared validation schemas.
- SSR/loader initial data.
- Client `EventSource` live updates.

Depends on: PR 7.

Acceptance criteria:

- The example follows TanStack Start server/client file separation.
- Secrets and SpacetimeDB server connections stay in server-only modules.

### PR 9: Effect Runtime Variant Or Cookbook

Branch: `examples/effect-gateway-runtime`

Purpose: Show how an Effect-based server can manage SpacetimeDB WSS resources and SSE fanout.

Scope:

- `ManagedRuntime` at framework edges.
- `Layer` for gateway, auth broker, event bus, and telemetry services.
- `Effect.acquireRelease` for WSS lifecycle.
- `Stream`, `Queue`, and `PubSub` for subscription fanout and backpressure.
- Runtime disposal on process shutdown.

Depends on: PR 7.

Acceptance criteria:

- This remains optional and does not make Effect a requirement for the generic SpacetimeDB gateway pattern.
- The example demonstrates lifecycle and backpressure, not a full product app.

### PR 10: Analog Variant

Branch: `examples/analog-gateway-sse`

Purpose: Add an Angular/Analog variant of the server gateway and SSE relay example.

Scope:

- Analog API/server route for SSE.
- Analog API/server route for reducer calls.
- Server-side data fetching through page `.server.ts` load functions for initial snapshots.
- Server-only gateway module used by Nitro/h3 handlers.
- Angular service, signal, or RxJS adapter that consumes `EventSource`.
- Shared validation schemas for server route inputs.

Depends on: PR 7.

Acceptance criteria:

- The example follows Analog server/client separation.
- Secrets and SpacetimeDB server connections stay in server-side route/load code.
- The example shows that the gateway pattern works for Angular apps, not only React apps.

### PR 11: Multi-Tenant Authorization Cookbook

Branch: `docs/multi-tenant-authorization-cookbook`

Purpose: Document common SaaS auth schemas and reducer/view patterns.

Scope:

- `actor_identity_map`
- `tenant`
- `membership`
- `role`
- `role_permission`
- `session_context`
- `impersonation_grant`
- `api_key_grant`
- `audit_event`
- sender-filtered views
- reducer guards

Depends on: PR 3 and PR 4.

Acceptance criteria:

- The docs clearly explain what belongs in JWT claims versus SpacetimeDB tables.
- Mutable authorization state is modeled in tables, not long-lived claims.

### PR 12: Robot Actors, API Keys, And Integrations Guide

Branch: `docs/robot-actors-api-keys`

Purpose: Document service accounts, API-key exchange, and machine actors.

Scope:

- API-key validation by the app backend.
- Short-lived robot JWT minting.
- Robot subject naming.
- Delegated actor pattern.
- Reducer audit metadata.
- Rotation and revocation.

Depends on: PR 4 and PR 11.

Acceptance criteria:

- The guide distinguishes human, robot, and delegated actors.
- It makes clear that long-lived API keys should not be sent directly from browsers or used as SpacetimeDB bearer tokens.

### PR 13: Token Diagnostics CLI

Branch: `feat/token-diagnostics-cli`

Purpose: Add CLI support for debugging OIDC/JWT migrations.

Scope:

- Decode token claims.
- Verify token against JWKS.
- Check `iss`, `sub`, `aud`, `exp`, and signing algorithm.
- Derive SpacetimeDB identity from issuer and subject.
- Detect opaque OAuth tokens and explain that a JWT resource audience is needed.
- Generate local test keys or document how to generate them.

Depends on: PR 4 and maintainer approval for CLI surface.

Acceptance criteria:

- Auth migration failures become diagnosable without writing ad hoc scripts.
- Tests cover valid JWTs, expired JWTs, wrong audience, unsupported algorithms, bad JWKS, and opaque tokens.

### PR 14: TypeScript SDK Token Provider And Reconnect Ergonomics

Branch: `feat/ts-sdk-token-provider-reconnect`

Purpose: Improve SDK ergonomics if maintainer feedback confirms this belongs in the SDK rather than only docs/examples.

Scope:

- Token provider callback for reconnects.
- Optional reconnect helper that fetches a fresh token.
- Clear auth error classification.
- Documentation for token expiry and refresh behavior.

Depends on: PR 2 and maintainer approval.

Acceptance criteria:

- Apps do not need to hand-roll token refresh around every connection.
- The SDK behavior is explicit when a token expires or auth is rejected.

## Dependency Order

Recommended opening order:

1. PR 1 to get maintainer agreement on the architecture and questions.
2. PRs 2 and 3 in parallel after initial feedback.
3. PRs 4, 5, and 6 once SDK lifecycle and actor attribution language is settled.
4. PR 7 after the core docs are accepted.
5. PRs 8, 9, and 10 as optional example variants.
6. PRs 11 and 12 for deeper SaaS authorization guidance.
7. PRs 13 and 14 only after maintainers agree on CLI and SDK API surfaces.

## What Not To Combine

- Do not combine Better Auth docs with the generic auth broker guide. The generic guide should stay useful for any app-owned session system.
- Do not combine the SSE gateway example with the TanStack Start, Analog, or Effect variants. The generic example should stay framework-light.
- Do not combine Keycloak migration docs with Better Auth docs. OIDC identity migration applies to many providers.
- Do not combine CLI diagnostics with SDK reconnect changes. They have different reviewers and risk profiles.

## Minimal First Milestone

The smallest useful milestone is PRs 1 through 4:

1. Public architecture issue.
2. Server-side TypeScript SDK lifecycle docs.
3. Gateway actor attribution docs.
4. App-owned auth broker guide.

That set would let a SaaS team understand whether the architecture is supported and how to build the secure path. The later PRs add provider-specific guidance, examples, tooling, and higher-level authorization patterns.
