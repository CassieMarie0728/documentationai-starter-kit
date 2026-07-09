## Error handling

MCP Hub uses tRPC's built-in error handling, so failed requests return structured error codes that your client can inspect reliably. Most errors include a stable `code`, a human-readable `message`, and optional diagnostic data for validation failures, MCP server connectivity problems, or upstream faults.

Use the error `code` for branching logic in your client. Use `message` for logs or user-facing summaries, and inspect `data` when you need field-level validation details or connection-specific context.

## Error response format

Most failed requests follow the same shape:

```json
{
  "error": {
    "code": "UNPROCESSABLE_CONTENT",
    "message": "Invalid input",
    "data": {
      "issues": [
        {
          "path": ["serverId"],
          "message": "Required"
        }
      ]
    }
  }
}
```

In development environments, responses may also include a stack trace for debugging:

```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "Failed to execute tool",
    "data": {
      "serverId": "srv_9k2m1x",
      "tool": "search_docs"
    },
    "stack": "TRPCError: Failed to execute tool\n    at ..."
  }
}
```

### Fields

| Field | Type | Description |
| --- | --- | --- |
| `error.code` | string | Stable error identifier for client-side branching and retries. |
| `error.message` | string | Human-readable summary of the failure. |
| `error.data` | object | Optional diagnostic context such as validation issues, connection metadata, or upstream error details. |
| `error.stack` | string | Optional stack trace, typically returned only in development. Do not depend on it in production. |

## tRPC error codes

MCP Hub uses standard tRPC-style error codes for request, authorization, validation, and server failures.

| Code | HTTP status | Meaning |
| --- | --- | --- |
| `BAD_REQUEST` | 400 | Invalid input or malformed request. |
| `UNAUTHORIZED` | 401 | Missing or invalid authentication. |
| `FORBIDDEN` | 403 | Authenticated, but not allowed to perform the action. |
| `NOT_FOUND` | 404 | Requested resource does not exist. |
| `METHOD_NOT_SUPPORTED` | 405 | HTTP method is not allowed for the route. |
| `TIMEOUT` | 408 | Request did not complete in time. |
| `CONFLICT` | 409 | Request conflicts with existing state, such as a duplicate name. |
| `PRECONDITION_FAILED` | 412 | Required condition was not met before the request ran. |
| `PAYLOAD_TOO_LARGE` | 413 | Request body exceeded the allowed size. |
| `UNPROCESSABLE_CONTENT` | 422 | Validation failed. |
| `TOO_MANY_REQUESTS` | 429 | Rate limit exceeded. |
| `INTERNAL_SERVER_ERROR` | 500 | Unexpected server-side failure. |
| `NOT_IMPLEMENTED` | 501 | Feature exists in the API surface but is not available yet. |
| `BAD_GATEWAY` | 502 | Upstream service returned an invalid or failed response. |
| `SERVICE_UNAVAILABLE` | 503 | Service is temporarily unavailable. |

## Common error scenarios

Use this table to map common failures to likely fixes.

| Scenario | Code | Typical cause | How to fix |
| --- | --- | --- | --- |
| Missing token | `UNAUTHORIZED` | Authorization header is absent or malformed. | Send a valid bearer token in the `Authorization` header. |
| Expired or invalid token | `UNAUTHORIZED` | Token expired, was revoked, or cannot be verified. | Refresh the token or sign in again. |
| Insufficient permissions | `FORBIDDEN` | Token is valid but lacks required role or scope. | Grant the required permissions or use a token with the correct access. |
| Unknown server or tool | `NOT_FOUND` | `serverId`, tool name, or referenced resource does not exist. | Verify identifiers and confirm the resource exists in the target environment. |
| Wrong HTTP method | `METHOD_NOT_SUPPORTED` | Client called the endpoint with an unsupported method. | Use the expected method for the route. |
| Duplicate resource name | `CONFLICT` | Resource already exists with the same unique identifier or name. | Rename the resource or fetch the existing one before creating another. |
| Missing required condition | `PRECONDITION_FAILED` | Request depends on state that is not satisfied. | Check prerequisites and retry only after the required condition is true. |
| Request body too large | `PAYLOAD_TOO_LARGE` | Arguments or payload exceeded size limits. | Reduce payload size, split the request, or send only required fields. |
| Schema validation failed | `UNPROCESSABLE_CONTENT` | Zod rejected one or more fields. | Inspect field-level issues in `error.data.issues` and correct the payload. |
| Request burst exceeded limits | `TOO_MANY_REQUESTS` | Client sent too many requests in a short period. | Back off, respect retry headers, and reduce burst traffic. |
| MCP server connection failed | `TIMEOUT` or `BAD_GATEWAY` | Upstream MCP server did not respond or failed during handshake. | Check server availability, network access, and transport configuration. |
| Unsupported feature | `NOT_IMPLEMENTED` | API route or capability is not available yet. | Remove the call or gate it behind feature detection until support is added. |
| Temporary outage | `SERVICE_UNAVAILABLE` | MCP Hub or a dependency is temporarily down. | Retry with backoff and monitor service health. |

## MCP server connection errors

MCP Hub surfaces connection errors through the server router when it cannot establish or maintain a valid MCP server session. These errors usually happen before a tool runs.

Common connection failure modes include:

- Connection refused
- Connection timeout
- Authentication failure
- Protocol mismatch

A connection refusal usually means the target host or port is wrong, the server is not running, or a firewall blocked access. A timeout usually means the server is reachable but did not respond quickly enough.

Authentication failures happen when MCP Hub reaches the server but the server rejects the provided credentials. Protocol mismatches happen when the client and server connect successfully but do not agree on MCP expectations or version semantics.

Example connection error:

```json
{
  "error": {
    "code": "TIMEOUT",
    "message": "Failed to connect to MCP server",
    "data": {
      "reason": "connection timeout",
      "serverId": "srv_9k2m1x",
      "transport": "http"
    }
  }
}
```

Example protocol mismatch:

```json
{
  "error": {
    "code": "BAD_REQUEST",
    "message": "Protocol mismatch",
    "data": {
      "expectedProtocol": "2024-11-25",
      "receivedProtocol": "2024-08-01"
    }
  }
}
```

## Validation errors

MCP Hub uses Zod schemas to validate procedure input. When validation fails, the API returns `UNPROCESSABLE_CONTENT` with field-level issues in `error.data`.

A typical validation response looks like this:

```json
{
  "error": {
    "code": "UNPROCESSABLE_CONTENT",
    "message": "Input validation failed",
    "data": {
      "issues": [
        {
          "path": ["arguments", "query"],
          "message": "Expected string, received number"
        }
      ]
    }
  }
}
```

Each issue points to the failing field path and explains why validation failed. Use these details to highlight invalid fields in your client or to log actionable diagnostics during development.

## Rate limit errors

Rate limiting returns `TOO_MANY_REQUESTS` when your client exceeds the allowed request volume. Treat rate limits as temporary failures and retry with backoff instead of retrying immediately.

A rate limit response may include headers such as:

- `Retry-After`
- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset`

If `Retry-After` is present, wait that many seconds before retrying. If it is absent, use exponential backoff with jitter to avoid synchronized retries.

Example:

```json
{
  "error": {
    "code": "TOO_MANY_REQUESTS",
    "message": "Rate limit exceeded",
    "data": {
      "scope": "api"
    }
  }
}
```

## Handling errors client-side

Handle MCP Hub errors in `try`/`catch` blocks and branch on `error.code`. Do not rely on parsing error messages, because `code` is the stable contract for client behavior.

For validation failures, inspect `error.data.issues`. For retryable failures such as `TIMEOUT`, `TOO_MANY_REQUESTS`, or `SERVICE_UNAVAILABLE`, use bounded retries with backoff.

### JavaScript

```javascript
async function executeTool() {
  try {
    const response = await fetch("https://api.mcphub.local/trpc/tools.execute", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer mcp_example_9xmk7q2r"
      },
      body: JSON.stringify({
        serverId: "srv_9k2m1x",
        tool: "search_docs",
        arguments: {
          query: "rate limits"
        }
      })
    });

    const payload = await response.json();

    if (!response.ok) {
      const err = payload.error || {};

      switch (err.code) {
        case "UNAUTHORIZED":
          throw new Error("Authentication failed. Sign in again.");
        case "FORBIDDEN":
          throw new Error("Your token does not have access to this action.");
        case "UNPROCESSABLE_CONTENT":
          console.error("Validation issues:", err.data?.issues || []);
          throw new Error(err.message || "Validation failed.");
        case "TOO_MANY_REQUESTS":
          throw new Error("Rate limited. Retry after a short delay.");
        case "TIMEOUT":
        case "SERVICE_UNAVAILABLE":
          throw new Error("Temporary upstream failure. Retry with backoff.");
        default:
          throw new Error(err.message || "Unexpected MCP Hub error.");
      }
    }

    return payload;
  } catch (error) {
    console.error("Tool execution failed:", error);
    throw error;
  }
}
```

### TypeScript

```typescript
type McpHubErrorResponse = {
  error?: {
    code?: string;
    message?: string;
    data?: {
      issues?: Array<{
        path: Array<string | number>;
        message: string;
      }>;
      [key: string]: unknown;
    };
    stack?: string;
  };
};

async function executeTool(): Promise<unknown> {
  const response = await fetch("https://api.mcphub.local/trpc/tools.execute", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": "Bearer mcp_example_9xmk7q2r"
    },
    body: JSON.stringify({
      serverId: "srv_9k2m1x",
      tool: "search_docs",
      arguments: {
        query: "protocol mismatch"
      }
    })
  });

  const payload = (await response.json()) as McpHubErrorResponse;

  if (response.ok) {
    return payload;
  }

  const code = payload.error?.code;
  const message = payload.error?.message ?? "Request failed";

  if (code === "UNPROCESSABLE_CONTENT") {
    console.error("Validation issues:", payload.error?.data?.issues ?? []);
  }

  if (code === "TOO_MANY_REQUESTS") {
    const retryAfter = response.headers.get("retry-after");
    throw new Error(
      retryAfter
        ? `Rate limited. Retry after ${retryAfter} seconds.`
        : "Rate limited. Retry later."
    );
  }

  if (code === "TIMEOUT" || code === "SERVICE_UNAVAILABLE") {
    throw new Error("Temporary service failure. Retry with backoff.");
  }

  throw new Error(`${code ?? "UNKNOWN_ERROR"}: ${message}`);
}
```

## tRPC error utilities

On the server, throw `TRPCError` from procedures to return structured error responses that clients can handle consistently.

Example:

```typescript
import { TRPCError } from "@trpc/server";
import { z } from "zod";

const executeToolInput = z.object({
  serverId: z.string().min(1),
  tool: z.string().min(1),
  arguments: z.record(z.unknown())
});

async function executeToolProcedure(input: z.infer<typeof executeToolInput>) {
  const server = await getServerById(input.serverId);

  if (!server) {
    throw new TRPCError({
      code: "NOT_FOUND",
      message: "MCP server not found"
    });
  }

  if (!server.isConnected) {
    throw new TRPCError({
      code: "TIMEOUT",
      message: "MCP server is unavailable"
    });
  }

  try {
    return await server.executeTool(input.tool, input.arguments);
  } catch (error) {
    throw new TRPCError({
      code: "BAD_GATEWAY",
      message: "Upstream MCP server failed to execute the tool",
      cause: error
    });
  }
}
```

Use `TRPCError` when you want the client to receive a predictable code and message. Let Zod handle input validation so invalid payloads produce `UNPROCESSABLE_CONTENT` with field-level details automatically.

## Debugging guidance

Check `error.code` before anything else. That value tells you whether to retry, re-authenticate, fix the request payload, or investigate an upstream dependency.

Inspect `error.data` before retrying. Validation issues, missing permissions, or protocol mismatch details usually mean the request is not retryable until you change it.

If you receive `INTERNAL_SERVER_ERROR`, `BAD_GATEWAY`, or repeated `TIMEOUT` responses, correlate the client error with server-side logs. Structured logs are usually the fastest way to identify upstream failures, handshake issues, or unhandled exceptions.