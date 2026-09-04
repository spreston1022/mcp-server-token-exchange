# Notes: token exchange on a standalone MCP Server

This repo exposes a single MCP server route, `/mcp-server-delegated`
(`mcpServerHandler`), that performs a real per-user token exchange
(RFC 8693) before calling its downstream API: the caller's token is
exchanged for a new token scoped to the downstream API, rather than the
caller's original token being forwarded as-is. It uses Keycloak for this
because it needs Custom Token Exchange, which requires an Auth0 plan tier
this tenant doesn't have.

## Why this route doesn't use Zuplo's built-in DCR policies

Zuplo's DCR-based OAuth policies (`mcp-*-oauth-inbound`) issue their own
session token to the caller and handle the provider token internally --
route code downstream doesn't get access to the caller's original token.
Performing a real token exchange requires that original token, so this
route uses direct JWT validation against Keycloak instead
(`OpenIdJwtInboundPolicy`, `modules/token-exchange.ts`).

To still support clients that only know how to authenticate via OAuth
discovery (claude.ai's connector UI, for example), the route publishes its
own RFC 9728 Protected Resource Metadata document
(`modules/mcp-server-delegated-prm.ts`, served at
`/internal/mcp-server-delegated/prm`) and returns a
`WWW-Authenticate: Bearer resource_metadata="..."` header on unauthenticated
requests (`modules/require-bearer-with-prm-hint.ts`) so clients can discover
it. This points directly at Keycloak's own OAuth endpoints, so clients
register and log in against Keycloak itself.

This metadata document lives at a project-specific path rather than the
conventional `/.well-known/oauth-protected-resource/mcp-server-delegated`
path: `OpenIdJwtInboundPolicy`'s own `oAuthResourceMetadataEnabled` option
routes through the same shared metadata mechanism `McpGatewayPlugin` uses
for its own DCR-based routes, which only serves the routes it manages
itself -- so a route using plain JWT validation needs to publish its
metadata document elsewhere.

## Demo environment

Token exchange in `/mcp-server-delegated` is tested against a local
Keycloak instance (Docker), exposed via a temporary public tunnel
(`cloudflared`) so it's reachable during testing. This setup is for local
testing only:

- The tunnel URL is not stable -- it changes if the container or tunnel
  restarts, and the relevant `KEYCLOAK_*` environment variables need
  updating when that happens.
- The realm's client-registration policies are configured for open testing
  (any host can register a client, any scope can be requested), and the
  downstream-audience client scope is realm-default rather than scoped to a
  specific trusted client. A real deployment would restrict registration to
  known redirect URIs, restrict allowed scopes explicitly, and scope the
  audience mapping to the specific client(s) that should be allowed to
  perform the exchange.
- Access tokens are set to a 1-hour lifetime for convenience during manual
  testing, longer than a typical default.

None of this affects the token-exchange logic itself (`token-exchange.ts`),
which works the same way against any RFC 8693-compliant token endpoint --
these are realm settings, not application code.
