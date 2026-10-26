## What is MCP?
Mcp stands for model context protocol. MCP is an open standard that defines how AI models and external systems communicate. You can think of it as an API that’s designed specifically for AI assistants. 

If an API lets two software programs exchange data, MCP allows an AI model to talk to your system. This opens up endless possibilities.

You could build an MCP server that lets ChatGPT read files from your local machine, or one that calls your company’s internal APIs to fetch data. You could even expose your own Python functions so that a model can use them as tools.

MCP makes this communication structured, secure, and extendable. It runs on familiar web technologies such as Server-Sent Events, or SSE, which allow the server to send real-time data streams to the client.
## Why Build your own mcp server?
Building an MCP server gives you control and flexibility.

You can connect AI models directly to your databases or internal systems, automate repetitive actions, and decide exactly what data an AI model can access.

It also allows you to experiment quickly. You can start small with a few simple tools and expand later into complex workflows.

By creating your own MCP server, you’re not just writing code – you’re defining how intelligent systems interact with the real world through your logic and data.

---

## Building a Demo MCP Server (Learning Log)

I followed a blog post to learn MCP servers, then built and tested my own small server to make the concepts concrete. This section is the build log: what I made, what broke, how I fixed it, and how I verified it worked.

> [!info] Project repository
> **GitHub:** [TheMoeaegon/mcp-server](https://github.com/TheMoeaegon/mcp-server)
>
> A minimal FastMCP server with three demo tools, SSE transport, and end-to-end testing docs.

### What I built

A single-file MCP server (`server.py`) that exposes three tools:

| Tool | Signature | What it does |
| --- | --- | --- |
| `add` | `add(a: int, b: int) -> int` | Adds two integers. |
| `get_secret_word` | `get_secret_word() -> str` | Returns a random word (`apple`, `banana`, `cherry`). |
| `get_current_weather` | `get_current_weather(city: str) -> str` | Fetches a text weather report for a city from `wttr.in`. |

Stack used: Python 3.14, [FastMCP](https://gofastmcp.com) 4.1.3, MCP SDK 2.3.0, `requests`, and `uv` for dependency management.

The core of the server is small:

```python
from fastmcp import FastMCP

port = int(os.environ.get("PORT", "8080"))
mcp = FastMCP(name)

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

FastMCP derives the tool name, description, and JSON schema from the function name, docstring, and type hints, so no manual schema wiring is needed.

The process starts with:

```python
mcp.run(transport="sse", port=port)
```

### Things that broke (and why)

The first version used the older MCP Python SDK style: `from mcp.server.fastmcp import FastMCP`. That failed with the installed stack. Three separate issues showed up:

**1. Wrong import path**

`mcp.server.fastmcp` no longer provides the FastMCP class in MCP SDK v2. FastMCP now ships as its own top-level package:

```python
from fastmcp import FastMCP
```

**2. Removed constructor arguments**

FastMCP v4 rejects `logger=` and `port=` passed to the constructor:

```text
TypeError: FastMCP() no longer accepts `port`. Pass `port` to `run_http_async()`, or set FASTMCP_PORT.
```

Fix: configure logging with Python's own `logging` module, and pass the port at run time instead.

**3. `FASTMCP_PORT` was read too late**

Setting `FASTMCP_PORT` *after* `import fastmcp` had no effect, because FastMCP loads its settings at import time. The server silently bound to the default port `8000` instead of `8080`, so requests to `localhost:8080` failed to connect.

Fix: resolve the port once and pass it explicitly:

```python
port = int(os.environ.get("PORT", "8080"))
...
mcp.run(transport="sse", port=port)
```

> [!tip] Lesson
> In FastMCP 4, runtime concerns (port, log level, transport) belong on `run()`, not the constructor. Environment variables must be set *before* importing `fastmcp` if you rely on them.

### SSE vs Streamable HTTP

`transport="sse"` starts the older HTTP+SSE transport, which uses two endpoints:

| Endpoint | Direction | Purpose |
| --- | --- | --- |
| `GET /sse` | server → client | Long-lived event stream |
| `POST /messages/?session_id=...` | client → server | JSON-RPC requests |

This is why the server URL is `http://localhost:8080/sse` rather than just the host and port. The server sends an `endpoint` event over `/sse` telling the client where to POST messages.

The modern alternative is **Streamable HTTP**, which uses a single endpoint (default `/mcp`):

```python
mcp.run(transport="http", port=port)
```

For new servers, Streamable HTTP is generally the better choice; SSE is kept for compatibility with older clients.

### How I tested it

Testing was done at three levels: static inspection, over the real SSE transport with the FastMCP CLI, and in-process against the `FastMCP` object.

**Static inspection (no running server)**

```bash
uv run fastmcp inspect server.py
```

```text
Server
  Name:         demo-mcp-server
  Version:      4.1.3
  Generation:   2

Components
  Tools:        3
  Prompts:      0
  Resources:    0
  Templates:    0
```

**Tool discovery over SSE**

```bash
uv run fastmcp list http://localhost:8080/sse -t sse
```

```text
Tools (3)

  add(a: int, b: int) -> dict
    Add two numbers

  get_secret_word() -> dict
    Get a random secret word

  get_current_weather(city: str) -> dict
    Get current weather for a city
```

**Calling tools**

```bash
uv run fastmcp call http://localhost:8080/sse add a=2 b=3 -t sse
# { "result": 5 }

uv run fastmcp call http://localhost:8080/sse get_secret_word -t sse
# { "result": "cherry" }

uv run fastmcp call http://localhost:8080/sse get_current_weather city=London -t sse
# { "result": "Weather report: London ... Sunny ... +12(7) °C ..." }
```

The `-t sse` flag is a **client-side** setting. It forces the CLI to speak SSE; otherwise it assumes Streamable HTTP for `http(s)://` URLs unless the URL already ends in `/sse`.

**In-process client (no port, no Uvicorn)**

```python
import asyncio
from fastmcp import Client
import server

async def main():
    async with Client(server.mcp) as c:
        print([t.name for t in await c.list_tools()])
        r = await c.call_tool("add", {"a": 2, "b": 3})
        print(r.data, r.is_error)

asyncio.run(main())
```

```text
tools: ['add', 'get_secret_word', 'get_current_weather']
data: 5
is_error: False
```

**Interactive inspection**

```bash
uv run fastmcp dev inspector server.py
```

This opens the browser-based MCP Inspector, useful for clicking through tools and inspecting raw requests/responses.

### Test results

| Check | Result |
| --- | --- |
| Tool registry (`fastmcp inspect`) | 3 tools, 0 prompts/resources |
| Discovery (`fastmcp list -t sse`) | All 3 tools listed with schemas |
| Argument passing (`add a=2 b=3`) | `5` |
| No-argument tool (`get_secret_word`) | `"cherry"` (random) |
| External API tool (`get_current_weather city=London`) | Weather text report |
| In-process client | `data=5`, `is_error=False` |

### Key takeaways

- MCP tools are just typed Python functions; FastMCP handles discovery, schema generation, and dispatch.
- Runtime configuration (transport, port, log level) is passed to `run()`, not the constructor.
- Transport choice changes the URL and the client flag: SSE → `/sse` + `-t sse`; Streamable HTTP → `/mcp`.
- Testing at the protocol level (CLI/Inspector) catches serialization and schema issues that unit-testing the raw functions would miss.
- Graceful error handling inside tools (returning an error string instead of raising) keeps the server responsive when a third-party API fails.

### Next steps

- Add a `/health` route with `@mcp.custom_route` for deployment checks.
- Try the Streamable HTTP transport and compare client behavior.
- Add automated tests with `pytest` + `fastmcp.Client`.
- Add auth before exposing the server beyond localhost.

---

Back to the repository: [TheMoeaegon/mcp-server](https://github.com/TheMoeaegon/mcp-server)
