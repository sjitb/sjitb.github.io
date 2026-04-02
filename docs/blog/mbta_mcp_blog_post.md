---
layout: page
title: Talking to Trains
permalink: /blog/talking-to-trains/
parent: Blog
nav_order: 20
---

# Talking to Trains: Building an MCP Server for Real-Time MBTA Data

There is a particular kind of frustration that comes from standing on a commuter rail platform, refreshing a transit app, and still not knowing whether your train is actually coming. The data exists. The APIs are public. And yet getting a clear, contextual answer, "the next Franklin Line train is 8 minutes out, but there is a moderate delay downstream," still requires you to do the cognitive assembly yourself.

That friction is exactly the kind of problem I want to solve as AI assistants get smarter. Not with a custom app. With a general-purpose agent that just knows how to ask the right questions.

This post is about building that: a real-time MBTA data layer, exposed as an MCP server, so Claude can answer natural-language questions about trains on your behalf.

---

## Why MCP Is the Interesting Part

If you have not encountered [Model Context Protocol](https://modelcontextprotocol.io) yet, here is the one-sentence version: it is an open standard that lets AI agents discover and call external tools at runtime, without those tools being hardcoded into the model or the host app.

That distinction matters more than it might seem.

Most tool use in LLM applications today is tightly coupled: a developer decides ahead of time which functions the model can call, wires them in, and ships. It works, but it does not scale. Every new capability requires a new integration, a new deployment, and a new round of prompt tuning.

MCP flips that. Tools advertise themselves: their name, their inputs, and their purpose. A compliant agent can discover and invoke them dynamically. The model does not need to be retrained for each capability. It reads the tool description and determines when to use it.

This is the early shape of something bigger: a world where APIs do not just serve applications, they serve agents.

---

## The Architecture

The project has two layers with a clean separation.

Layer 1, `mbta_client.py`, is an async wrapper around the [MBTA V3 REST API](https://api-v3.mbta.com). It handles authentication and request lifecycle, and returns parsed API data.

Layer 2, `mbta_mcp_server.py`, sits on top of the client and exposes named tools Claude can call.

```text
Claude (Claude Code / Claude Desktop)
        |  MCP protocol over stdio
        v
  mbta_mcp_server.py   <- tools Claude can discover and call
        |  Python function calls
        v
  mbta_client.py       <- async wrapper around MBTA V3 API
        |  HTTPS + API key
        v
  api-v3.mbta.com      <- live transit data
```

The `stdio` transport is key: Claude Code or Claude Desktop spawns the server as a subprocess and communicates over standard input/output. No HTTP server and no ports to manage.

---

## Quickstart in 60 Seconds

1. Clone the repo and install dependencies: `pip install -r requirements.txt`
2. Create `config.ini` in the repo root:

```ini
[mbta]
api_key = YOUR_MBTA_API_KEY
```

3. Make sure `.mcp.json` points to `mbta_mcp_server.py`
4. Restart Claude Code (or Claude Desktop, if using desktop config)
5. Ask: "Any delays on the Red Line right now?"

---

## What the MBTA API Actually Offers

The MBTA V3 API is richer than most people realize. Beyond static schedule data, it provides:

- Real-time predictions (live departure estimates)
- Alerts (service disruptions with severity and effect)
- Vehicle positions
- Route and stop metadata

The API follows JSON:API with `data` and optional `included` resources. Predictions and schedules are separate endpoints. Predictions represent what is happening now; schedules represent what was planned.

Route type filters are numeric:

- `0`: light rail
- `1`: heavy rail
- `2`: commuter rail

A client method from this project looks like this:

```python
async def get_predictions(
    self,
    stop_id: str,
    route_id: str | None = None,
    direction_id: int | None = None,
) -> list[dict[str, Any]]:
    params = {
        "filter[stop]": stop_id,
        "filter[route]": route_id,
        "filter[direction_id]": direction_id,
        "include": "route,trip,stop",
        "sort": "departure_time",
    }
    data = await self._get("/predictions", params)
    return data.get("data", [])
```

---

## Wrapping It as MCP Tools

The MCP server exposes five tools:

- `get_line_alerts(route_id)`
- `get_predictions(stop_id, route_id, direction_id)`
- `get_route_status(route_id)`
- `find_stop(name)`
- `get_stops_for_route(route_id)`

Tool descriptions matter because Claude reads them to decide when to invoke each tool.

```python
@mcp.tool()
async def get_line_alerts(route_id: str) -> str:
    """Return active service alerts for an MBTA route."""
    async with MBTAClient() as client:
        alerts = await client.get_alerts(route_id=route_id)
    # format and return a human-readable summary
```

For Claude Code, the repo config is minimal:

```json
{
  "mcpServers": {
    "mbta": {
      "command": "python",
      "args": ["mbta_mcp_server.py"],
      "cwd": "${workspaceFolder}"
    }
  }
}
```

Important implementation note: this project currently reads the API key from `config.ini` (not from MCP env vars).

---

## What It Unlocks

Once the server is running, interactions like these work naturally:

- "Any delays on the Red Line right now?"
- "When is the next train from South Station toward Needham?"
- "Is the Green Line running normally?"

Claude decides which tools to call based on tool descriptions and query context.

If a question has directional ambiguity, Claude may need an extra call (for example, adding `direction_id` after resolving stop and route context).

### Example Claude Code interaction

User:

> Next train from Park Street on the Red Line?

Likely tool sequence:

1. `find_stop("Park Street")`
2. `get_predictions(stop_id="place-pktrm", route_id="Red")`

Claude response style:

> Next Red Line departures from Park Street are at 5:42 PM and 5:49 PM. Current service status is on time.

---

## Limitations

- Real-time data quality depends on MBTA feed availability.
- Some station names map to multiple stop IDs and need disambiguation.
- Prediction availability can vary by route and time of day.

---

## Where This Goes Next

A few natural extensions:

- Multi-stop trip planning with a higher-level tool like `get_trip_status(from_stop, to_stop)`
- Proactive monitoring agents for commuter lines
- Voice or chat integrations reusing the same MCP server
- Cross-agency expansion to other GTFS-RT compatible systems

---

## A Closing Thought on Agentic AI

The compelling part is not just MBTA integration. It is the pattern.

Much of the most useful AI work now happens at the boundary between models and external systems. MCP is one way to formalize that boundary so capabilities are discoverable and composable instead of hardcoded case by case.

An MCP server for MBTA trains is a small, concrete example. The pattern scales well beyond transit.

---

Built with the [MBTA V3 API](https://api-v3.mbta.com), the [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk), and Claude.

Project source: https://github.com/sjitb/mbta-schedule-app
