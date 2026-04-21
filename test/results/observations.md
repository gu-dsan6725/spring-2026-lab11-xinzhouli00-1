# Lab 11: A2A Agent Discovery — Observations

## 1. A2A Messages Exchanged

From the test output, two HTTP POST requests were made to `127.0.0.1:10001` (Travel Assistant):

```
POST http://127.0.0.1:10001  →  200 OK (2774 bytes)   # Test 1: flight search
POST http://127.0.0.1:10001  →  200 OK (5920 bytes)   # Test 2: booking workflow
```

The second response is ~2x larger, which makes sense — it includes the discovery step and the delegated booking attempt on top of the original flight search.

*Note: full JSON-RPC message bodies were not captured in the debug output.*

---

## 2. How the Travel Assistant Discovered the Flight Booking Agent

The Travel Assistant has a `discover_remote_agents` skill that queries the registry
at `/api/discover-agents` with a natural language query. The workflow was:

1. User asks to book a flight
2. Travel Assistant calls `discover_remote_agents` → POST to registry:
   ```
   POST http://127.0.0.1:10001/api/discover-agents?query=book+flights
   ```
3. Registry returns matching agents with metadata (see Section 3)
4. Agent card is cached via `view_cached_remote_agents`
5. Travel Assistant calls `invoke_remote_agent` to delegate the booking

The registry does semantic matching — the query `"book flights"` returned the
Flight Booking Agent with a `relevance_score` of 0.95.

---

## 3. JSON-RPC Request/Response Format

**Discovery request/response**:

Request:
```
POST http://127.0.0.1:10001/api/discover-agents?query=book+flights
```

Response:
```json
{
  "query": "book flights",
  "agents_found": 1,
  "agents": [{
    "name": "Flight Booking Agent",
    "description": "Flight booking and reservation management agent",
    "url": "http://127.0.0.1:10002",
    "tags": ["booking", "flights", "reservations"],
    "trust_level": "verified",
    "visibility": "public",
    "relevance_score": 0.95,
    "skills": [
      {"id": "check_availability", ...},
      {"id": "reserve_flight", ...},
      {"id": "confirm_booking", ...},
      {"id": "process_payment", ...},
      {"id": "manage_reservation", ...}
    ]
  }]
}
```

**Agent invocation**:
```json
{
  "jsonrpc": "2.0",
  "method": "invoke",
  "params": {
    "skill": "reserve_flight",
    "message": "Reserve 1 seat on flight NYC-LAX on Dec 20"
  },
  "id": 1
}
```

---

## 4. Agent Card Contents and How They Were Used

Both cards follow the same schema (`protocolVersion: 0.3.0`):

| Field | Travel Assistant | Flight Booking Agent |
|---|---|---|
| `url` | `127.0.0.1:10001` | `127.0.0.1:10002` |
| `transport` | JSONRPC | JSONRPC |
| `streaming` | true | true |
| `skills` | search, discover, invoke... | reserve, confirm, pay... |

The Travel Assistant used the Flight Booking Agent's card to:
- Find its endpoint URL (`10002`)
- Know which skills are available (e.g. `reserve_flight`, `confirm_booking`)
- Confirm they share the same protocol version and transport

Without the card, the Travel Assistant would have no way to know the booking agent exists or how to talk to it.

---

## 5. Benefits and Limitations

**Benefits**
- Agents are loosely coupled — Travel Assistant doesn't hard-code the booking agent's address, it discovers it at runtime
- Easy to extend: adding a new agent (e.g. hotel booking) just means registering a new card, no code changes needed in other agents
- Skills are self-describing, so the LLM can decide which one to call based on natural language

**Limitations**
- The booking agent was unavailable in the test (`"Flight Booking Agent didn't respond"`), and the system had no retry or fallback — it just reported failure
- Discovery adds latency; in the test, Test 2 took ~24s vs ~7s for Test 1
- No authentication between agents — any agent that knows the URL can invoke another, which is a security concern in production
