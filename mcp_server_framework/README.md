# MCP Server Framework

A user-friendly, structured MCP (Model Context Protocol) server example for Melt. It handles JSON-RPC 2.0 over stdio (one message per line), centralizes parsing and errors, and makes it easy to add new methods.

## Structure

```
mcp_server_framework/
├── main.melt       # Entry: loads config + lib + router, runs MCP loop
├── config.melt     # Config (e.g. debug logging)
├── lib/
│   └── rpc.melt    # JSON-RPC helpers: success(), error(), standard error codes
├── router.melt     # McpRouter: parse request → dispatch by method → send response
└── README.md
```

## How it works

1. **main.melt** imports config, RPC helpers, and the router, then calls `setMcpHandler("McpRouter")` and `runMcp()`.
2. **runMcp()** (built-in) reads one line from stdin, sets it as the request, calls `McpRouter().handle()`, then writes the response line to stdout. This repeats until stdin is closed.
3. **McpRouter.handle()** gets the raw line, decodes JSON, reads `id` and `method`, and calls `dispatch(method, params, id)`.
4. **dispatch()** routes to a handler (e.g. `ping`, `echo`, `add`, `tools/list`, `initialize`). Each handler returns a result object built with `success(id, result)` or `error(id, code, message)` from **lib/rpc.melt**.
5. Invalid JSON, missing method, or unknown method produce a JSON-RPC error response with the right code.

## Run

From the **project root** (so imports resolve):

```bash
# Single request
echo '{"jsonrpc":"2.0","id":1,"method":"ping"}' | ./bin/melt examples/mcp_server_framework/main.melt

# Multiple (one line per request; one line per response)
echo '{"jsonrpc":"2.0","id":1,"method":"ping"}
{"jsonrpc":"2.0","id":2,"method":"echo","params":{"message":"Hi"}}
{"jsonrpc":"2.0","id":3,"method":"add","params":{"a":10,"b":20}}' | ./bin/melt examples/mcp_server_framework/main.melt
```

## Built-in methods

| Method          | Description                    | Params / result                    |
|-----------------|--------------------------------|------------------------------------|
| `ping`          | Health check                   | Result: `"pong"`                   |
| `echo`          | Echo a message                 | `params.message` → result          |
| `add`           | Add two numbers                | `params.a`, `params.b` → result    |
| `tools/list`    | List available tools          | Result: array of `{name, description}` |
| `initialize`    | MCP-style init                 | Result: capabilities + serverInfo  |

## Config

In **config.melt**, set `MCP_DEBUG = true` to log each request and response to stderr.

## Adding a new method

1. In **router.melt**, add a branch in `dispatch()`:
   ```melt
   if (mname == "myMethod") {
       return this.myMethod(params, id);
   }
   ```
2. Implement the handler on `McpRouter`:
   ```melt
   method myMethod(params, id) {
       let x = params.x;   // use params as needed
       return success(id, "some result");
   }
   ```
3. For errors (e.g. invalid params), return `error(id, INVALID_PARAMS, "message")` (constants like `INVALID_PARAMS` are in **lib/rpc.melt**).

**Note:** Do not use the keyword `method` as a variable name; use `req["method"]` to read the method from the request, and use a different name (e.g. `mname`) for your variable.

## JSON-RPC error codes (lib/rpc.melt)

- `PARSE_ERROR` (-32700)
- `INVALID_REQUEST` (-32600)
- `METHOD_NOT_FOUND` (-32601)
- `INVALID_PARAMS` (-32602)
- `INTERNAL_ERROR` (-32603)

## Dependencies

Uses Melt built-ins: `getMcpRequest`, `setMcpResponse`, `jsonDecode`, `jsonEncode`, `objectCreate`, `arrayCreate`, `arrayPush`, `writeStderr`.
