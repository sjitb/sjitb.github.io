---
layout: page
title: Talking to Trains
permalink: /blog/talking-to-trains/
parent: Blog
nav_order: 20
---

# Talking to Trains: Building an MCP Server for Real-Time MBTA Data

This project started with a product question: how do we reduce rider decision latency when service conditions change in real time?

Transit apps expose data, but riders still do the synthesis. The objective was to move that synthesis into an agent workflow without losing traceability.

## Problem Statement

Riders ask goal-oriented questions:

- Is my line delayed?
- What is the next departure from this station?
- Which train gets me to a destination before a deadline?

The MBTA API provides the raw ingredients, but question-to-answer flow is multi-step and context-dependent. The goal was a tool layer an agent can execute reliably.

## Decision Log

### Decision 1: Separate transport integration from agent tooling

Decision:
Build two layers, an async MBTA client and an MCP server on top.

Rationale:
The MBTA integration lifecycle and the agent-tool lifecycle move at different speeds. Coupling them would increase risk and slow iteration.

Outcome:
API changes stay in the client while tool contracts evolve independently.

### Decision 2: Use MCP stdio transport instead of a hosted service

Decision:
Run tools through MCP over stdio as a local subprocess.

Rationale:
The priority was development speed and operational simplicity.

Tradeoff:
Stdio is excellent for local workflows, but it is not a direct replacement for a multi-tenant hosted tool platform.

Outcome:
Faster testing and lower development overhead.

### Decision 3: Favor narrow intent-level tools over broad meta-tools

Decision:
Expose focused tools for alerts, predictions, schedules, stop lookup, route stops, route status, and trip schedule confirmation.

Rationale:
Broad tools increase ambiguity. Narrow tools improve composability and observability.

Outcome:
More consistent sequencing and cleaner failure handling.

### Decision 4: Treat real-time and planned data as separate decision domains

Decision:
Use predictions for immediate departures and schedules plus trip-level validation for future-arrival planning.

Rationale:
"Next train now" and "arrive by 9 AM tomorrow" are different planning problems with different data guarantees.

Outcome:
Lower risk of confident but incorrect future-trip answers.

## Technical Learnings

### Learning 1: Tool descriptions are part of system behavior

In MCP systems, descriptions are runtime orchestration guidance. Precision in tool semantics directly affected invocation quality.

### Learning 2: Ambiguity is the default case

Stop names, direction assumptions, and line naming regularly introduced ambiguity. Designing for disambiguation early was more valuable than optimizing for a single-call path.

### Learning 3: Multi-step reasoning should be explicit

For arrival-deadline questions, the reliable pattern was:

1. Resolve origin and destination stops.
2. Pull candidate schedules constrained by time window.
3. Validate each candidate trip against destination stop timing.

This sequence improved correctness and explainability.

## Operational Usage Patterns

To validate usability beyond demos, I tracked common query patterns and expected orchestration paths.

### Basic queries that should work consistently

- "Check route status for Red Line"
- "Show line alerts for CR-Franklin"
- "Find stops matching South Station"
- "List stops for Orange Line"
- "Show tomorrow's inbound schedule from West Natick on the Worcester line"

### Multi-step patterns used as design checks

Example 1: next train from a named station

1. User asks: "When is the next Red Line train from Park Street?"
2. Agent resolves stop: find_stop("Park Street")
3. Agent fetches real-time departures: get_predictions(stop_id="place-pktrm", route_id="Red")
4. Agent returns a readable departure summary.

Example 2: commuter rail status plus departures

1. User asks: "Is Needham service delayed and what are the next departures?"
2. Agent checks service state: get_line_alerts("CR-Needham")
3. Agent resolves stop and fetches predictions when needed.
4. Agent combines disruption context with departure timing.

Example 3: future trip planning with arrival deadline

1. User asks: "If I need to reach South Station by 9am tomorrow, which train do I take from West Natick on the Worcester or Framingham line?"
2. Agent resolves origin and destination stop IDs.
3. Agent queries planned service window: get_schedules(stop_id="place-WNatick", route_id="CR-Worcester", date="2026-04-03", direction_id=1, max_time="09:00")
4. Agent validates candidate trips using get_trip_schedule(trip_id=...) at destination level.
5. Agent reports the latest feasible departure that still meets the deadline.

These patterns became acceptance criteria. If orchestration failed a step, tool boundaries or descriptions needed refinement.

### Prompting guidance that improved reliability

- Include route IDs when possible: Red, Orange, CR-Franklin, Green-D.
- Include stop context when useful: for example, "at South Station".
- Ask for bounded output when useful: for example, "show top 5 only".

### Route ID conventions worth documenting

Subway IDs commonly used:

- Red, Orange, Blue
- Green, Green-B, Green-C, Green-D, Green-E
- Mattapan

Commuter rail IDs commonly used:

- CR-Franklin, CR-Needham, CR-Providence
- CR-Worcester (also covers Framingham line stops such as West Natick and Framingham)

## Takeaways

- Treat tool interfaces as product surfaces, not implementation details.
- Optimize for debuggable orchestration, not only endpoint coverage.
- Separate fast-changing intent logic from slower integration plumbing.
- Define correctness boundaries early, especially where real-time and planned data diverge.

## Next Decisions Under Consideration

- Add confidence signals tied to feed freshness and data completeness.
- Improve automatic disambiguation strategy for near-duplicate stop names.
- Introduce a higher-level trip-planning orchestrator that preserves step-level explainability.

The broader takeaway: agentic systems work best when each layer has a clear contract. Data retrieval, tool semantics, and orchestration logic should be independently understandable and evolvable.

Built with the [MBTA V3 API](https://api-v3.mbta.com), the [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk), and Claude.

Project source: https://github.com/sjitb/mbta-schedule-app
