## Discover, execute, and manage tools from connected MCP servers.

## Navigating to the Tools tab

Select the **Tools** tab from the bottom tab bar. You need at least one connected server before tools are available. If you have not connected one yet, see [Managing Servers](./managing-servers.md).

<Callout kind="info">

Tools only appear for servers that are currently connected. If the Tools tab is empty, check the connection status first.

</Callout>

## Selecting a server

If you have multiple servers connected, use the server selector at the top of the Tools tab to choose which server to inspect. The tool list updates to show only the tools exposed by the selected server.

A server can expose a different catalog of tools, schemas, and permissions. If you switch servers and a tool disappears, that usually means the selected server does not provide it.

## Discovering tools

When a server connects, MCP Hub calls `tools/list` to fetch the available tools. Refresh the list any time by pulling down on the tool list or tapping **Refresh**.

The request uses the `tools/list` JSON-RPC method and supports pagination through the `cursor` parameter.

```json
{
  "jsonrpc": "2.0",
  "id": "req_tools_list_01",
  "method": "tools/list",
  "params": {
    "cursor": null
  }
}
```

A successful response returns a list of tool definitions and, when more results exist, a `nextCursor` value.

```json
{
  "jsonrpc": "2.0",
  "id": "req_tools_list_01",
  "result": {
    "tools": [
      {
        "name": "list_repositories",
        "description": "List repositories available to the authenticated user",
        "inputSchema": {
          "type": "object",
          "properties": {
            "visibility": {
              "type": "string"
            }
          }
        }
      }
    ],
    "nextCursor": "page_2"
  }
}
```

> If a server exposes many tools, MCP Hub may need multiple `tools/list` calls to load the full catalog.

## Understanding tool schemas

Each tool includes a schema that tells MCP Hub what arguments the tool accepts and how to validate them before execution.

<ParamField body="name" param-type="string" required="true" show-location="false">

The tool's unique identifier. MCP Hub sends this value in the `tools/call` request.

</ParamField>

<ParamField body="description" param-type="string" required="false" show-location="false">

A human-readable summary of what the tool does.

</ParamField>

<ParamField body="inputSchema" param-type="object" required="false" show-location="false">

A JSON Schema object that defines accepted arguments, expected data types, and required fields.

</ParamField>

Example tool schema:

```json
{
  "name": "create_issue",
  "description": "Create an issue in a repository",
  "inputSchema": {
    "type": "object",
    "properties": {
      "owner": {
        "type": "string"
      },
      "repo": {
        "type": "string"
      },
      "title": {
        "type": "string"
      },
      "body": {
        "type": "string"
      }
    },
    "required": ["owner", "repo", "title"]
  }
}
```

## Filling in parameters

MCP Hub generates an input form from the tool's `inputSchema`. Required fields are marked with an asterisk, and optional fields can be left blank.

The generated form maps common schema types to these inputs:

| Type | Form input |
| --- | --- |
| `string` | Text field |
| `number` | Numeric input |
| `boolean` | Toggle switch |
| `array` | List editor |
| `object` | Nested form or JSON editor |

After you fill in the required fields, tap **Execute** to run the tool.

> If a field expects a number or boolean, enter that type directly instead of converting it to text. Schema validation usually fails before the request is sent.

## Executing tools

Executing a tool sends a `tools/call` JSON-RPC request to the selected server. MCP Hub uses the selected tool's `name` and the values you entered in the form to build the `arguments` object.

```json
{
  "jsonrpc": "2.0",
  "id": "req_tool_call_01",
  "method": "tools/call",
  "params": {
    "name": "create_issue",
    "arguments": {
      "owner": "acme-inc",
      "repo": "platform",
      "title": "Fix broken webhook retries",
      "body": "Webhook deliveries are not retrying after transient failures."
    }
  }
}
```

The server returns tool output as content blocks.

```json
{
  "jsonrpc": "2.0",
  "id": "req_tool_call_01",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Issue created successfully: acme-inc/platform#184"
      }
    ]
  }
}
```

If execution succeeds, the result viewer opens with the tool output. That result confirms the request reached the server and the server completed the operation.

## Viewing results

MCP Hub formats tool responses in a result viewer so you can inspect output without reading raw JSON for every run.

| Result type | How it displays |
| --- | --- |
| `text` | Rendered as formatted text |
| `json` | Rendered with syntax highlighting |
| `image` | Rendered inline as an image |
| `resource` | Rendered as a link or embedded content |

The result viewer also shows execution metadata such as status and execution time. Use that metadata to confirm whether the tool finished successfully or failed before producing output.

## Execution history

Past tool executions appear in the **Analytics** tab. Each entry includes the tool name, server, timestamp, status, and duration.

Execution history is useful when you need to compare runs, confirm whether a tool was triggered, or inspect patterns in failures over time.

## Troubleshooting

<ExpandableGroup>
  <Expandable title="Tool not found">

  The server may have removed or renamed the tool. Refresh the catalog from the Tools tab first.

  If the tool still does not appear, clear the tool cache from the server detail view and check that the server still exposes the tool.

  </Expandable>
  <Expandable title="Invalid parameters">

  Check the tool's `inputSchema` for required fields and expected types. The error message usually identifies the field that failed validation.

  If a parameter expects a number, boolean, array, or object, make sure the value matches that type exactly.

  </Expandable>
  <Expandable title="Server timeout">

  A timeout usually means the server is slow, overloaded, or disconnected during execution. Check the server connection status in the Servers tab.

  If the server stays connected but the tool keeps timing out, the tool may need more time than the current execution timeout allows.

  </Expandable>
  <Expandable title="Permission denied">

  Your API key or OAuth token may not include the scopes required by that tool. Re-authenticate from the server detail view to refresh credentials.

  If the error continues after re-authentication, check the server-side permission model for that tool.

  </Expandable>
</ExpandableGroup>