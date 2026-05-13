# GitHub Issue Draft: Server-Side Realtime Gateway, App-Owned Auth, And SSE Relay

## Summary

We are evaluating SpacetimeDB for a family of multi-tenant SaaS applications where the web application owns authentication, organization membership, admin flows, API keys, and customer SSO, while SpacetimeDB owns realtime state, reducers, subscriptions, audit-friendly writes, and identity-bound domain data.

The architecture we want to standardize is:

1. The browser talks to a full-stack application framework such as TanStack Start for React or Analog for Angular.
2. Better Auth owns the user's web session, organizations, admin features, SSO, and API keys.
3. The application server connects to SpacetimeDB over WSS using the generated TypeScript/Node SDK.
4. The application server subscribes to SpacetimeDB tables/views, calls reducers, and maintains a server-side realtime projection.
5. The browser receives live updates from the application server over SSE.
6. Browser mutations go through framework-native server functions, API routes, or server handlers, where auth, tenant context, input validation, and rate limits are enforced before reducers are called.

This is not a request for SpacetimeDB to become a web-session provider. The request is for first-class docs and SDK support for teams that keep app auth in their web stack and use SpacetimeDB as the realtime data plane behind that server boundary.

The main design decision we need guidance on is actor attribution. When the browser does not connect directly, the gateway needs a documented way to choose between per-user SpacetimeDB connections, service-account connections with delegated actor context, or a hybrid topology.

## Why This Architecture

The common SpacetimeDB browser-client pattern is powerful, but some SaaS teams need a server-side gateway for operational and security reasons:

- Keep long-lived credentials, API keys, refresh tokens, and robot credentials off the browser.
- Centralize tenant selection, impersonation, organization membership, customer portal boundaries, and mutable authorization in one application server layer.
- Replace an existing Keycloak-style identity system without forcing every browser client and integration to understand database-specific tokens.
- Avoid one SpacetimeDB WSS connection per browser tab when a server-side fanout model is more appropriate.
- Use SSE for browser delivery because it fits dashboards, back-office tools, document workflows, mobile web, and infrastructure that already handles HTTP streams well.
- Keep writes disciplined: server functions validate input, derive actor context, call reducers, and then the UI updates through the realtime stream.

## Planned Stack

- Full-stack web framework options:
  - TanStack Start for React, SSR, server functions, API routes, and route-level auth boundaries.
  - Analog for Angular, file-based routing, server-side data fetching, API/server routes, hybrid SSR/SSG, and Nitro/h3-powered server integration.
- Better Auth for app-owned authentication, organizations, admin flows, customer SSO, API keys, and OAuth/OIDC capabilities.
- Effect v4 on the server for runtime structure:
  - `ManagedRuntime` bridges framework handlers to long-lived Effect services.
  - `Layer` owns shared services such as the SpacetimeDB connection manager, auth broker, event bus, and telemetry.
  - `Effect.acquireRelease` and scopes manage WSS connection lifecycle and cleanup.
  - `Stream`, `Queue`, and `PubSub` model subscription events, fanout, backpressure, replay, and SSE delivery.
  - `Schema` validates server-function inputs and typed event envelopes.
- SpacetimeDB generated TypeScript/Node SDK from the server side, connected over WSS.
- SSE from the application server to the browser for live UI updates.

## Framework Shape

The requested pattern should not be locked to one frontend framework. The important boundary is server-side ownership of the SpacetimeDB connection and browser delivery over normal HTTP streams.

| Concern | TanStack Start | Analog |
| --- | --- | --- |
| UI framework | React | Angular |
| Initial page data | Route loaders and SSR | Server-side data fetching through page `.server.ts` load functions |
| Browser mutations | Server functions or API routes | API/server routes backed by Nitro/h3 handlers |
| Live browser updates | API route that serves SSE to `EventSource` | API/server route that serves SSE to `EventSource`, Angular service, signal, or RxJS adapter |
| Server-only code | `.server.ts`, server functions, API routes | `.server.ts` load files and `src/server/routes/api` handlers |
| Gateway runtime | Shared service used by handlers | Shared service used by Nitro/h3 handlers |
| Test target | Server functions, API routes, CLI commands | API/server routes, server load code, CLI commands |

In both variants, the browser should not need SpacetimeDB credentials. It should receive initial data through SSR/server-side data fetching and incremental updates through SSE, while writes travel through validated server code that calls SpacetimeDB reducers.

## Why Testability Matters

We want this architecture to be easy to test from an application monorepo without opening a browser. The same gateway code should be callable from TanStack Start handlers, Analog API/server routes, scheduled jobs, migration scripts, CI smoke tests, and local CLI commands. That makes the SpacetimeDB integration easier to trust before it is wired into a full UI.

Effect v4 is useful here because the gateway can be structured as small services with explicit dependencies: auth broker, token verifier, SpacetimeDB connection manager, subscription projector, SSE event bus, clock, telemetry, and persistence. In tests, those dependencies can be swapped with Layers, test clocks, in-memory PubSubs, fake JWKS, fake Better Auth sessions, or local SpacetimeDB connections. Long-lived resources such as WSS connections and SSE streams can be acquired and released deterministically.

The CLI use case is important because many real workflows are operational, not purely browser-driven:

- Mint a SpacetimeDB access token from a test session or robot credential.
- Connect the server gateway to a local or staging database.
- Subscribe to a view and assert that the projected event stream contains expected rows.
- Call reducers through the same server-side command path used by web mutations.
- Assert that an SSE envelope is emitted with the expected `event`, `id`, `tenant_id`, `schema_version`, and `payload`.
- Simulate token expiry, reconnect, resubscribe, and catch-up behavior.
- Dry-run issuer/subject identity migration and verify identity-link table updates.
- Run smoke tests in CI against local SpacetimeDB before deploying an application server.

SpacetimeDB docs and examples would be stronger if they showed how to test the server gateway as a first-class runtime component. The goal is not to require Effect, but to make sure the documented architecture supports deterministic tests, CLI-invoked workflows, and CI automation for auth, subscriptions, reducers, and SSE relay behavior.

## Request: Make This A Documented First-Class Pattern

### 1. Server-Side TypeScript SDK Guidance

Please document whether the generated TypeScript SDK is intended to be used from long-running Node/server runtimes as a first-class client, not only from browsers.

Topics that would help:

- Recommended Node runtime versions and WebSocket implementations.
- How to keep a long-lived `DbConnection` alive in a server process.
- How to reconnect, resubscribe, and rehydrate server-side projections.
- How to refresh auth tokens on a server-side WSS connection.
- How to call reducers from request handlers while subscription updates are flowing on the same connection.
- Whether to run one connection per process, tenant, database, user context, or workload.
- How to preserve native `ctx.sender` semantics when reducers are called from a server-side gateway on behalf of a browser user.
- How concurrent reducer calls are ordered and correlated with subscription updates.
- How to shut down cleanly in serverless, container, and long-lived process environments.

### 2. Gateway Actor Attribution And Connection Topologies

Please document the recommended topology for server-side gateways:

- Per-user connection: the gateway opens or pools a SpacetimeDB connection with a user-scoped JWT. Reducers see the user as `ctx.sender`. This preserves the native identity model but may create many WSS connections.
- Service connection: the gateway uses a service or robot actor connection and passes an effective actor context to reducers. This reduces WSS fanout but needs explicit guidance so apps do not accidentally trust client-supplied actor IDs.
- Hybrid: the gateway uses service connections for subscriptions and projections, but user-scoped connections or short-lived calls for writes that need native sender attribution.

The docs should explain which approach SpacetimeDB recommends for SaaS apps and how each approach affects authorization, audit trails, reducer APIs, subscription filtering, connection counts, token refresh, and reconnect behavior.

### 3. Server Gateway And SSE Relay Example

An official example would be valuable:

```text
Browser
  |
  | HTTP, server functions/API routes, EventSource/SSE
  v
Full-stack app server
  | TanStack Start / Analog / similar
  |
  | Better Auth session, tenant context, API key validation
  | Effect runtime, PubSub, Stream, telemetry
  | SpacetimeDB generated TypeScript SDK
  v
SpacetimeDB over WSS
```

The example should show:

- SSR or loader-based initial data.
- SSE endpoint for live updates.
- Server-side SpacetimeDB subscriptions feeding an Effect `PubSub`.
- Event envelopes with `event`, `id`, `tenant_id`, `schema_version`, and `payload`.
- Browser reconnect using `Last-Event-ID`.
- Snapshot plus incremental-update behavior.
- Backpressure behavior when browsers are slow.
- Authorization changes that revoke or narrow an SSE stream.
- Reducer calls from validated server functions.
- An explicit actor-attribution strategy for reducer calls.
- CLI-invoked smoke tests that exercise token minting, reducer calls, subscription projection, and SSE event output without a browser.
- Clean process shutdown that closes SpacetimeDB connections and SSE streams.

### 4. App-Owned Auth Broker Documentation

Please add a documented pattern for:

```text
Better Auth session -> app server authorization checks -> short-lived SpacetimeDB JWT
```

Recommended claim contract:

- `iss`: app-owned issuer.
- `sub`: stable app actor ID, not email.
- `aud`: SpacetimeDB database, app, or resource audience.
- `azp`: calling client where useful.
- `token_type`: for example `spacetime-access`.
- `sid`: app session ID for audit and revocation checks.
- `tenant_id`: active tenant or organization context.
- `actor_ref`: stable application actor reference.
- `scope` or `perms`: compact permission hints only.
- `membership_version`: optional stale-token invalidation number.

The docs should emphasize that this token is not the web session. It is a short-lived database access token derived from a verified app session.

### 5. Better Auth Specific Guide

Better Auth has two relevant paths that SpacetimeDB docs could explain.

Custom broker mode:

- Better Auth owns the browser session.
- The app server verifies the session and organization membership.
- A small server route signs a short-lived SpacetimeDB JWT.
- The route exposes JWKS or uses a Better Auth JWT plugin JWKS endpoint.

OAuth/OIDC provider mode:

- Better Auth's OAuth provider package exposes OIDC metadata, JWKS, introspection, revocation, authorization-code flows, refresh tokens, and custom access-token claims.
- Clients must request a `resource` audience for SpacetimeDB so Better Auth returns a JWT access token.
- Without a `resource` audience, Better Auth can issue an opaque access token, which SpacetimeDB cannot directly validate through JWKS.

Specific documentation gaps:

- Which JWT algorithms SpacetimeDB supports. Better Auth's JWT plugin defaults to EdDSA, while ES256 or RS256 may be safer defaults until compatibility is explicit.
- How to configure `issuer`, `audience` or `validAudiences`, `expirationTime`, and `jwksPath`.
- How to expose `/.well-known/openid-configuration` and `/jwks` when auth is mounted under an app path such as `/api/auth`.
- How to project Better Auth organization membership into small JWT claims and keep mutable authorization in SpacetimeDB tables.

### 6. Keycloak Exit And Identity Migration Guide

Many teams adopting app-owned auth will be replacing an existing OIDC provider. Docs should cover:

- SpacetimeDB identity derivation from `iss` plus `sub`.
- Why changing issuer or subject changes the resulting SpacetimeDB identity.
- Identity-link tables that map SpacetimeDB identities and federated issuer/subject pairs to application users.
- Running old and new issuers in parallel during migration.
- Precomputing identity mappings before cutover.
- Retiring the old issuer only after audit and traffic checks show no active dependency.

### 7. Machine Actors, API Keys, And Integrations

SaaS apps need webhooks, scheduled jobs, importers, AI agents, MCP servers, and customer-built integrations. These should not impersonate browser users by default.

Recommended documentation:

- Validate long-lived API keys in the app backend.
- Exchange valid API keys for short-lived SpacetimeDB JWTs.
- Use stable robot subjects such as `robot:<integration_id>`.
- Carry tenant and permission hints as claims, then re-check mutable grants in SpacetimeDB tables.
- Distinguish direct robot access from delegated access:
  - Human: `sub=<user_id>`.
  - Robot: `sub=robot:<service_or_key_id>`.
  - Delegated: `sub=<user_id>` plus `act=<robot_or_admin_actor>`.

This maps well to Better Auth's API-key package, which supports hashed keys, prefix lookup, expiration, rate limits, permissions, metadata, and user or organization references.

### 8. Multi-Tenant Authorization Cookbook

Please add a cookbook for SaaS authorization that uses private tables, views, reducers, and server-side gateway checks together.

Useful examples:

- `actor_identity_map`
- `tenant`
- `membership`
- `role`
- `role_permission`
- `session_context`
- `impersonation_grant`
- `api_key_grant`
- `audit_event`
- sender-filtered views such as `my_profile`, `my_memberships`, `active_context`, `visible_documents`, and `visible_audit_events`

The guidance should explain what belongs in JWT claims versus SpacetimeDB tables. Mutable roles, permissions, tenant membership, impersonation grants, and revocation-sensitive state should live in tables. JWTs should carry only stable identity and small routing/context hints.

### 9. Token Diagnostics And Local Development

Auth migrations are hard to debug. A small CLI or documented workflow would save time:

- Decode token claims.
- Verify issuer, subject, audience, expiration, and algorithm.
- Verify a token against JWKS.
- Derive the SpacetimeDB `Identity` for an issuer and subject.
- Detect opaque OAuth tokens and recommend requesting a `resource` audience.
- Generate local ES256 or RS256 keys for development.
- Mint local test JWTs for module and client tests.
- Fail closed when a local issuer is used against production.

### 10. Effect And Full-Stack Framework Reference Notes

The example does not need to depend on Effect, TanStack Start, or Analog, but it would be useful for docs to acknowledge the shape of modern full-stack web apps:

- Framework-native server functions, API routes, or server handlers are the browser-to-server mutation boundary.
- Inputs are validated at the network boundary.
- Server-only code stays in server-only modules.
- Framework-safe wrappers can be imported by client code while secrets and database connections remain server-only.
- API routes are better for SSE, webhooks, and external integrations.
- SSR loaders provide initial data; SSE provides incremental updates.

For TanStack Start apps:

- Server functions are a good fit for mutations.
- API routes are a good fit for SSE streams and external integrations.
- Route loaders can provide the initial snapshot before the browser opens an `EventSource`.

For Analog apps:

- API/server routes are a good fit for reducer calls, SSE streams, webhooks, and integration endpoints.
- Server-side data fetching through page `.server.ts` files can provide the initial snapshot.
- Angular services, signals, or RxJS adapters can consume the browser `EventSource` stream.

For Effect-based apps, the ideal integration is:

- A single app runtime built from Layers.
- SpacetimeDB WSS connection managed as an acquired resource.
- Subscription callbacks turned into typed streams.
- Bounded `PubSub` for backpressure and optional replay.
- SSE streams derived from authorized event streams.
- CLI commands and CI tests that reuse the same services as request handlers.
- Runtime disposal on process shutdown.

## Concrete Use Cases

This architecture supports:

- Multi-tenant operator dashboards.
- Customer portals with restricted views over shared domain data.
- Document, workflow, and journal-style applications that need low-latency updates and audited writes.
- Admin impersonation with explicit grants and audit records.
- Customer-managed API keys and server-to-server integrations.
- AI agents or MCP servers acting as robot users with scoped reducer access.
- Migration away from a centralized OIDC provider while preserving identity continuity and auditability.

## Open Questions For Maintainers

1. Is long-running server-side use of the generated TypeScript SDK a supported and recommended deployment shape?
2. What connection topology do you recommend for a server-side gateway: per process, per tenant, per authenticated user, or per workload?
3. How should a gateway preserve `ctx.sender` for user-initiated reducer calls?
4. Can a connection refresh its token without a full reconnect, or should the gateway reconnect and resubscribe?
5. What ordering guarantees should a gateway expect between reducer results and subscription updates?
6. Are there recommended patterns for replay/catch-up after gateway downtime?
7. Which JWT algorithms are supported for OIDC/JWKS validation today?
8. Can docs include a Better Auth example alongside Auth0, Clerk, and generic OIDC?
9. Can examples include both React/TanStack Start and Angular/Analog variants, or at least keep the server gateway pattern framework-neutral?
10. Can examples include CLI-invoked tests that exercise the gateway without a browser?

## Suggested Priority

1. Document server-side TypeScript SDK support and connection lifecycle.
2. Add an app-owned auth broker guide with Better Auth examples.
3. Add a server gateway plus SSE relay example.
4. Add Keycloak/OIDC migration documentation.
5. Add token diagnostics for issuer, subject, audience, algorithm, and JWKS.
6. Add multi-tenant authorization and robot actor cookbooks.

SpacetimeDB already has the core primitives we need: reducers, subscriptions, OIDC/JWT identity, private tables, views, and generated clients. The main request is to make this server-side gateway architecture obvious, documented, and supported enough that SaaS teams can adopt it without inventing the auth and realtime bridge from scratch.
