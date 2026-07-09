## Integrations

MCP Hub connects to MCP servers over JSON-RPC 2.0, which gives you a consistent way to register servers, test connectivity, and discover available tools. Use built-in provider presets when you want a fast path for common services, or register a custom MCP server when you need to connect your own local or remote endpoint.

## Built-in provider integrations

MCP Hub includes preset integrations for common providers. These presets handle the provider identity and expected authentication model so you can validate credentials and register the server with fewer inputs.

| Provider | What it connects to | Example tools | Authentication |
| --- | --- | --- | --- |
| GitHub | GitHub repositories, issues, pull requests, and profile data | `list_repositories`, `create_issue`, `create_pull_request`, `list_issues`, `search_repositories`, `create_repository`, `get_user_profile` | Bearer token (PAT) |
| Slack | Slack workspaces, channels, and users | Post messages, manage channels, read users | API-Key (bot token) |
| Notion | Notion databases, pages, and workspace content | Query databases, create pages, manage workspace | API-Key (integration token) |

## Add a custom MCP server

Register a custom server when you already have an MCP-compatible endpoint and want MCP Hub to connect to it directly. The flow is always the same: prepare the server address, choose transport and auth, register the server, test the connection, then discover tools.

### Prepare your MCP server URL

Start with the address MCP Hub should use to reach your server. The URL format depends on the transport:

- Use a network URL for remote servers, such as `https://mcp.acme.dev/sse` or `wss://mcp.acme.dev/ws`
- Use a local command or local process entrypoint for `stdio` servers in the format your client expects
- Make sure the server is reachable from the environment where MCP Hub runs

A good first check is whether you can start or reach the server outside MCP Hub. If the server is not running or the URL is wrong, registration may succeed but connection testing will fail.

### Choose a transport type

Transport determines how MCP Hub communicates with the server.

| Transport | Best for | How it works | Notes |
| --- | --- | --- | --- |
| `stdio` | Local servers | MCP Hub communicates with a local process over standard input and output | Fastest option and does not require network access |
| `sse` | Remote servers over HTTP | MCP Hub receives streamed events over HTTP | Good default for remote deployments |
| `websocket` | Real-time remote servers | MCP Hub keeps a persistent bidirectional connection | Best when the server sends frequent updates or expects low-latency interaction |

### Choose an auth method

Authentication tells MCP Hub which header pattern to use when connecting to the server.

| Auth type | Header pattern | Best for | Notes |
| --- | --- | --- | --- |
| `bearer` | `Authorization: Bearer ...` | OAuth-style tokens and personal access tokens | Common for hosted APIs |
| `api-key` | API key header | Service tokens and bot tokens | Used by providers such as Slack and Notion |
| `basic` | `Authorization: Basic ...` | Username and password or service credentials | Use only when the server expects Basic auth |

### Register the server

Call `mcp.registerServer` with the server name, URL, connection type, auth type, and auth token. A successful registration returns a `serverId` that you use in later calls.

```ts
type RegisterServerInput = {
  name: string
  url: string
  connectionType: "stdio" | "sse" | "websocket"
  authType: "bearer" | "api-key" | "basic"
  authToken: string
}

async function registerServer(client: {
  callTool: (name: string, args: unknown) => Promise<unknown>
}) {
  const result = await client.callTool("mcp.registerServer", {
    name: "Acme MCP",
    url: "https://mcp.acme.dev/sse",
    connectionType: "sse",
    authType: "bearer",
    authToken: "ghp_example_pat_9xmk7q2r"
  } satisfies RegisterServerInput)

  return result
}
```

Expected response shape:

```ts
{
  serverId: "srv_7f3c2a91",
  status: "registered"
}
```

Once you have a `serverId`, verify that MCP Hub can actually connect before you try to use tools.

### Test the connection

Call `mcp.testConnection` with the `serverId` returned during registration. This tells you whether MCP Hub can reach the server and how long the round trip took.

```ts
async function testConnection(client: {
  callTool: (name: string, args: unknown) => Promise<unknown>
}) {
  const result = await client.callTool("mcp.testConnection", {
    serverId: "srv_7f3c2a91"
  })

  return result
}
```

Expected response shape:

```ts
{
  connected: true,
  latency: 84,
  error: null
}
```

If `connected` is `true`, move on to tool discovery. If it is `false`, inspect the `error` value first because most integration failures come from an unreachable URL, the wrong transport, or invalid credentials.

### Discover available tools

Call `mcp.discoverTools` after the connection succeeds. MCP Hub returns the tools exposed by that server, including each tool name, description, and input schema.

```ts
async function discoverTools(client: {
  callTool: (name: string, args: unknown) => Promise<unknown>
}) {
  const result = await client.callTool("mcp.discoverTools", {
    serverId: "srv_7f3c2a91"
  })

  return result
}
```

Expected response shape:

```ts
{
  tools: [
    {
      name: "search_documents",
      description: "Search internal knowledge base documents",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string" },
          limit: { type: "number" }
        },
        required: ["query"]
      }
    }
  ]
}
```

When tool discovery succeeds, the integration is ready to use. The returned tool names and schemas tell you exactly what operations the server exposes.

### Complete example

Use this end-to-end example when you want a single script that registers a server, verifies connectivity, and lists its tools.

```ts
type MCPClient = {
  callTool: (name: string, args: unknown) => Promise<any>
}

async function connectCustomServer(client: MCPClient) {
  const registration = await client.callTool("mcp.registerServer", {
    name: "Acme MCP",
    url: "https://mcp.acme.dev/sse",
    connectionType: "sse",
    authType: "bearer",
    authToken: "mcp_example_token_4fd92ab8"
  })

  console.log("Registered server:", registration.serverId, registration.status)

  const connection = await client.callTool("mcp.testConnection", {
    serverId: registration.serverId
  })

  if (!connection.connected) {
    throw new Error(`Connection failed: ${connection.error}`)
  }

  console.log("Connected in ms:", connection.latency)

  const tools = await client.callTool("mcp.discoverTools", {
    serverId: registration.serverId
  })

  console.log("Discovered tools:", tools.tools.map((tool: any) => tool.name))
}

connectCustomServer({
  async callTool(name, args) {
    console.log("Call tool:", name, args)
    return {}
  }
})
```

## Use the mcpServers namespace

Use the `mcpServers` namespace when you want to register a preset provider such as GitHub, Slack, or Notion. This flow is shorter than custom registration because the provider metadata already exists.

### List available preset providers

Call `mcpServers.getAvailableServers` to see which preset providers MCP Hub can register.

```ts
async function getAvailableServers(client: {
  callTool: (name: string, args?: unknown) => Promise<unknown>
}) {
  const result = await client.callTool("mcpServers.getAvailableServers")
  return result
}
```

Expected response shape:

```ts
[
  {
    provider: "github",
    displayName: "GitHub",
    description: "Connect GitHub repositories and issues",
    requiredAuth: "bearer"
  },
  {
    provider: "slack",
    displayName: "Slack",
    description: "Connect Slack channels and users",
    requiredAuth: "api-key"
  },
  {
    provider: "notion",
    displayName: "Notion",
    description: "Connect Notion databases and pages",
    requiredAuth: "api-key"
  }
]
```

### Validate the token before registering

Call `mcpServers.validateToken` before registration so you can catch auth errors early. This is especially useful in setup flows where you want to fail fast before storing or using the credential.

```ts
async function validateProviderToken(client: {
  callTool: (name: string, args: unknown) => Promise<unknown>
}) {
  const result = await client.callTool("mcpServers.validateToken", {
    provider: "github",
    token: "ghp_example_pat_9xmk7q2r"
  })

  return result
}
```

Expected response shape:

```ts
{
  valid: true,
  error: null
}
```

### Register the preset provider

Call `mcpServers.registerRealServer` after token validation succeeds. This registers the provider-backed server and returns the same kind of server identity used by the custom flow.

```ts
async function registerPresetProvider(client: {
  callTool: (name: string, args: unknown) => Promise<unknown>
}) {
  const result = await client.callTool("mcpServers.registerRealServer", {
    provider: "github",
    token: "ghp_example_pat_9xmk7q2r"
  })

  return result
}
```

Expected response shape:

```ts
{
  serverId: "srv_b82d1f50",
  status: "registered"
}
```

### Complete preset provider example

This example lists providers, validates a GitHub token, and registers the GitHub integration.

```ts
type MCPClient = {
  callTool: (name: string, args?: unknown) => Promise<any>
}

async function connectGitHubPreset(client: MCPClient) {
  const presets = await client.callTool("mcpServers.getAvailableServers")
  console.log("Available presets:", presets.map((server: any) => server.provider))

  const validation = await client.callTool("mcpServers.validateToken", {
    provider: "github",
    token: "ghp_example_pat_9xmk7q2r"
  })

  if (!validation.valid) {
    throw new Error(`Token validation failed: ${validation.error}`)
  }

  const registration = await client.callTool("mcpServers.registerRealServer", {
    provider: "github",
    token: "ghp_example_pat_9xmk7q2r"
  })

  console.log("Registered preset server:", registration.serverId, registration.status)
}

connectGitHubPreset({
  async callTool(name, args) {
    console.log("Call tool:", name, args)
    return []
  }
})
```

## Transport types explained

Choose the transport based on where the server runs and how it communicates.

| Transport | Deployment model | Performance | Network requirement | Best use case |
| --- | --- | --- | --- | --- |
| `stdio` | Local process | Fastest | No network | Local development and tightly coupled desktop or CLI workflows |
| `sse` | Remote HTTP endpoint | Moderate | HTTP required | Hosted MCP servers and simple remote integrations |
| `websocket` | Remote persistent endpoint | Low-latency after connect | Network required | Real-time or bidirectional workflows |

`stdio` is the simplest option when MCP Hub and the MCP server run in the same environment. `sse` works well for hosted services that expose HTTP streaming, while `websocket` fits servers that keep an active two-way session open.

## Auth methods explained

Choose the auth method that matches what the server expects. If the server documentation names a specific HTTP header pattern, use the matching MCP Hub auth type.

| Auth method | Typical header | Common use | Notes |
| --- | --- | --- | --- |
| `bearer` | `Authorization: Bearer token` | Personal access tokens and OAuth access tokens | Best for APIs that already use bearer auth |
| `api-key` | API key header | Bot tokens, service tokens, integration secrets | Common for provider-specific integrations |
| `basic` | `Authorization: Basic base64-credentials` | Username and password credentials | Prefer only when the server explicitly requires it |

## MCP protocol details

MCP Hub uses JSON-RPC 2.0 and targets protocol version `2024-11-25`. That means requests and responses follow the JSON-RPC structure while MCP-specific behavior is defined by the protocol version shared by the client and server.

At the protocol level, the core interaction pattern is:

- Initialize the session
- List available tools
- Call a tool with validated input

The corresponding MCP methods are typically named `initialize`, `tools/list`, and `tools/call`. MCP Hub exposes higher-level integration methods such as `mcp.registerServer`, `mcp.testConnection`, and `mcp.discoverTools`, but the connected server still speaks the MCP protocol underneath.

## Troubleshooting integrations

Most integration failures come from one of three areas: transport mismatch, invalid authentication, or a server that is reachable but not speaking MCP correctly. Start with connection testing before you debug tool behavior.

### Connection test fails

If `mcp.testConnection` returns `connected: false`, check these first:

- Confirm the server URL is correct and reachable from the MCP Hub runtime
- Confirm the selected `connectionType` matches the server transport
- Confirm the auth token is valid and the selected `authType` matches the expected header format
- Confirm the server is running and accepting connections

### No tools are discovered

If `mcp.discoverTools` returns an empty tool list or fails:

- Confirm the server completed MCP initialization successfully
- Confirm the server implements tool discovery
- Confirm the connection test succeeds before running discovery
- Check whether the authenticated identity has permission to access tools

### Preset provider registration fails

If a built-in provider such as GitHub or Slack fails to register:

- Run `mcpServers.getAvailableServers` to confirm the provider name
- Run `mcpServers.validateToken` before registration
- Confirm you are using the correct token type for that provider
- Retry registration only after validation succeeds

### Protocol-level errors

If the server connects but behaves unexpectedly, inspect the MCP implementation itself.

- Confirm the server speaks JSON-RPC 2.0
- Confirm the server targets protocol version `2024-11-25`
- Confirm the server supports the expected method flow: `initialize`, `tools/list`, and `tools/call`

A successful integration usually looks like this: registration returns a `serverId`, connection testing returns `connected: true`, and tool discovery returns one or more tools. If one of those checkpoints fails, fix that stage before moving to the next one.