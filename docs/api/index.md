## API overview

MCP Hub exposes a type-safe API built on tRPC v11 at `/api/trpc`. You use it to manage MCP servers, discover and execute tools, handle tokens and webhooks, and access analytics through a single strongly validated interface.

The API is designed for application-to-application use inside the MCP Hub stack. Input validation runs through Zod on every procedure, which means invalid requests fail early and consistently before they reach business logic.

> Use `http://localhost:3000/api/trpc` as the development base URL.

## Base URL

Use this base URL in local development:

```text
http://localhost:3000/api/trpc
```

MCP Hub also exposes a small set of non-tRPC HTTP endpoints for health checks and AI-related routes. Those endpoints sit alongside the main tRPC API rather than replacing it.

## API architecture

The API combines a typed RPC layer with a conventional HTTP server and real-time transport where needed.

| Layer | Technology | What it does |
| --- | --- | --- |
| RPC API | tRPC v11 | Exposes namespaced procedures at `/api/trpc` with end-to-end type inference |
| HTTP server | Express | Serves the API and supporting HTTP endpoints |
| Validation | Zod | Validates procedure inputs on every request |
| Real-time | Socket.IO | Supports real-time communication where live updates are needed |

This architecture gives you strongly typed procedure calls in TypeScript while keeping the server surface straightforward to inspect and operate. If you are building against the API from a TypeScript client, type inference is one of the main benefits because request and response shapes stay aligned with the server definitions.

## API surface summary

The main tRPC API is organized into namespaces. Each namespace groups related procedures so you can find the right entry point quickly.

| Namespace | Purpose |
| --- | --- |
| `system` | Health and system information procedures |
| `auth` | Session-related procedures such as retrieving the current user and logging out |
| `oauth` | OAuth token exchange and related authorization flows |
| `mcp` | Core MCP server lifecycle, tool discovery, execution, testing, status, and cache management |
| `mcpServers` | Server catalog, token validation, server registration, tool discovery, execution, connection testing, and unregister flows |
| `tokens` | Token creation, update, deletion, lookup, and expiry management |
| `webhooks` | Webhook creation, update, deletion, and delivery handling |
| `analytics` | Execution analytics, macro analytics, and trending data |

A few namespaces cover similar concepts from different angles. In practice, `mcp` focuses on the main MCP Hub workflow, while `mcpServers` covers server-specific registration and execution operations.

## HTTP endpoints

Not every route in MCP Hub uses tRPC. These HTTP endpoints are also part of the API surface.

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/` | Root endpoint |
| `GET` | `/api/health` | Health check endpoint that returns `ok`, `timestamp`, and `version` |
| `POST` | `/api/ai/*` | AI-related HTTP endpoints |

Use `/api/health` for monitoring and readiness checks. A healthy response includes a JSON payload with the service status, current timestamp, and version.

## Quick example

Call a tRPC procedure by posting to the procedure path under `/api/trpc`. The exact payload shape depends on the procedure, but the pattern is consistent across namespaces.

```ts
const response = await fetch("http://localhost:3000/api/trpc/system.health", {
  method: "POST",
  headers: {
    "content-type": "application/json",
  },
  body: JSON.stringify({
    input: null,
  }),
});

if (!response.ok) {
  throw new Error(`Request failed with status ${response.status}`);
}

const result = await response.json();
console.log(result);
```

If the procedure exists and the request passes validation, the server returns a JSON response from the tRPC handler. For local testing, a successful health-style call confirms that the API server is running and that the procedure path is reachable.

## Authentication overview

MCP Hub uses session-based authentication with JWT cookies. That model lets browser-based clients authenticate through a session while the server continues to enforce access control on protected procedures.

For setup details, login flow behavior, and protected route expectations, see [Authentication](./authentication.md).

## Rate limiting overview

MCP Hub applies rate limits at both the global and API levels.

| Scope | Limit |
| --- | --- |
| Global | `1000` requests per `15` minutes |
| API | `100` requests per minute |

These limits help protect the service from bursts and abuse while keeping normal API usage predictable. For enforcement details and planning guidance, see [Rate limits](./rate-limits.md).

## Related API pages

- [API documentation](./api_documentation.md)
- [Authentication](./authentication.md)
- [Integrations](./integrations.md)
- [Errors](./errors.md)
- [Rate limits](./rate-limits.md)